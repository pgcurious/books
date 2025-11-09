# Inventing File I/O from First Principles
## The Journey from Memory to Persistent Storage

---

## Introduction: A World Without Files

Imagine you're building the first computer. You have:
- **RAM**: Fast memory that loses everything when you turn off the power
- **A program**: Running beautifully, processing data
- **A problem**: Everything disappears when you shut down

Your boss walks in: "Hey, can you save the user's work so they don't have to start over tomorrow?"

You stare at your RAM. It's volatile. It forgets everything.

**This book is about solving that problem.** Not by memorizing Java's `FileInputStream`, `BufferedReader`, or `Files.readAllLines()` — but by *inventing* them ourselves, one problem at a time.

By the end, you won't just know **how** to use these classes. You'll understand **why they exist** in the first place.

Let's start with nothing and build everything.

---

## Chapter 1: The Persistence Problem

### The Crisis

```java
// Your program
int userScore = 9000;
String userName = "Alice";
List<String> achievements = Arrays.asList("First Login", "Level 10");

// User runs program → Everything works!
// User closes program → GONE. All of it.
// User restarts → userScore is 0, userName is null, achievements is empty
```

**The Problem**: RAM is **volatile**. Power off = data death.

**The Constraint**: We need something **non-volatile** to store data permanently.

### Enter: The Hard Disk

Your hardware engineer shows you a disk drive:
- **Persistent**: Data survives power loss
- **Large**: Gigabytes of storage (way more than RAM)
- **Slow**: 1000x slower than RAM, but better than nothing

**The Question**: How do we get data from RAM to disk?

---

## Chapter 2: Attempt 1 — Write Raw Bytes

### The Simplest Solution

Your hardware gives you a primitive operation:

```java
// Hypothetical low-level API
void writeByteToDisk(long address, byte value);
byte readByteFromDisk(long address);
```

You can write one byte at a time to a disk address. Let's use it!

```java
// Save user score (int = 4 bytes)
int userScore = 9000;

writeByteToDisk(0, (byte)(userScore >>> 24));  // Write byte 1
writeByteToDisk(1, (byte)(userScore >>> 16));  // Write byte 2
writeByteToDisk(2, (byte)(userScore >>> 8));   // Write byte 3
writeByteToDisk(3, (byte)(userScore));         // Write byte 4

// Read it back
byte b1 = readByteFromDisk(0);
byte b2 = readByteFromDisk(1);
byte b3 = readByteFromDisk(2);
byte b4 = readByteFromDisk(3);

int restored = (b1 << 24) | (b2 << 16) | (b3 << 8) | b4;
```

### The Problem

**Writing is tedious**:
- Manual byte manipulation
- Tracking disk addresses manually
- Error-prone (wrong bit shift = corrupted data)

**What about complex data?**

```java
String userName = "Alice";  // How many bytes? Which addresses?
List<String> achievements;  // How do we even represent this?
```

### The Insight

**We need an abstraction**. Instead of thinking "bytes at address X," we want to think "write this data somewhere."

**Question**: Can we create a helper that manages the byte-level details?

---

## Chapter 3: Attempt 2 — The OutputStream Abstraction

### The Idea

What if we create an abstraction that:
1. Handles byte-level writing
2. Manages disk addresses automatically
3. Lets us write data sequentially

```java
class OutputStream {
    private long currentPosition = 0;

    void write(byte b) {
        writeByteToDisk(currentPosition, b);
        currentPosition++;
    }

    void write(byte[] bytes) {
        for (byte b : bytes) {
            write(b);
        }
    }
}
```

### Using It

```java
OutputStream out = new OutputStream();

// Write an integer
int userScore = 9000;
out.write((byte)(userScore >>> 24));
out.write((byte)(userScore >>> 16));
out.write((byte)(userScore >>> 8));
out.write((byte)(userScore));

// Write a string
String userName = "Alice";
byte[] nameBytes = userName.getBytes();
out.write(nameBytes);
```

**Better!** We're abstracting away address management.

### But...

**Still tedious**. We're manually converting ints to bytes. Can we automate that?

---

## Chapter 4: Attempt 3 — DataOutputStream (Typed Writing)

### The Idea

Let's add methods for writing common types directly:

```java
class DataOutputStream {
    private OutputStream out;

    void writeInt(int value) {
        out.write((byte)(value >>> 24));
        out.write((byte)(value >>> 16));
        out.write((byte)(value >>> 8));
        out.write((byte)(value));
    }

    void writeUTF(String text) {
        byte[] bytes = text.getBytes("UTF-8");
        writeInt(bytes.length);  // Write length first
        out.write(bytes);        // Then write the bytes
    }

    void writeBoolean(boolean b) {
        out.write(b ? 1 : 0);
    }
}
```

### Using It

```java
DataOutputStream data = new DataOutputStream();

data.writeInt(9000);           // Handles byte conversion automatically!
data.writeUTF("Alice");        // Handles string encoding!
data.writeBoolean(true);       // Simple!
```

**Much cleaner!** We've abstracted type serialization.

### This is Real!

**Java's actual `DataOutputStream` works exactly like this:**

```java
FileOutputStream fos = new FileOutputStream("save.dat");
DataOutputStream dos = new DataOutputStream(fos);

dos.writeInt(9000);
dos.writeUTF("Alice");
dos.writeBoolean(true);

dos.close();
```

**We just invented it from scratch.**

---

## Chapter 5: The Performance Crisis

### The Problem

Let's measure performance:

```java
// Write 1 million integers
for (int i = 0; i < 1_000_000; i++) {
    dos.writeInt(i);
}
```

**What happens?**

Each `writeInt()` calls `write()` four times. Each `write()` makes a **disk write** system call.

- 1 million integers = 4 million disk writes
- Each disk write: ~100 microseconds
- **Total time: 400 seconds (6.6 minutes!)**

For comparison, writing to RAM would take ~40 milliseconds.

**We're 10,000x slower than we need to be!**

### Why So Slow?

**System call overhead**:
- Switching from user mode to kernel mode is expensive
- Each disk write involves:
  - Context switch
  - Hardware interaction
  - Cache invalidation

**The Insight**: Disk I/O is slow. Making 4 million tiny writes is catastrophically slow.

### The Solution: Buffering

**Idea**: Accumulate data in memory (a **buffer**), then write it all at once.

```java
class BufferedOutputStream {
    private OutputStream out;
    private byte[] buffer = new byte[8192];  // 8KB buffer
    private int position = 0;

    void write(byte b) {
        buffer[position++] = b;

        if (position == buffer.length) {
            flush();  // Buffer full, write to disk
        }
    }

    void flush() {
        if (position > 0) {
            out.write(buffer, 0, position);  // One big write!
            position = 0;
        }
    }

    void close() {
        flush();  // Write remaining data
        out.close();
    }
}
```

### Performance Impact

**Before (unbuffered)**:
- 1 million integers = 4 million disk writes
- Time: 400 seconds

**After (buffered with 8KB buffer)**:
- 1 million integers = ~500 disk writes (4MB / 8KB)
- Time: 0.05 seconds

**We're 8,000x faster!**

### Real Java API

```java
FileOutputStream fos = new FileOutputStream("data.bin");
BufferedOutputStream bos = new BufferedOutputStream(fos);
DataOutputStream dos = new DataOutputStream(bos);

// Now writeInt() is fast!
for (int i = 0; i < 1_000_000; i++) {
    dos.writeInt(i);
}

dos.close();  // Don't forget! Flushes remaining buffer.
```

**This is why `BufferedOutputStream` exists!**

---

## Chapter 6: The Character Problem

### The New Requirement

Your boss: "Users want to save text files, not binary data."

You: "Easy! We have `DataOutputStream.writeUTF()`."

Boss: "No, I mean readable text files. Like `.txt` files. Users should be able to open them in Notepad."

### The Problem

```java
// Using DataOutputStream
dos.writeUTF("Hello");

// File contents (hex):
// 00 05 48 65 6C 6C 6F
//  ^    ^--- "Hello" in UTF-8
//  |--- Length prefix (2 bytes)
```

**Issue**: The length prefix makes it a binary format. Text editors won't understand it.

### What We Want

```
File contents:
48 65 6C 6C 6F  (just "Hello" in UTF-8, no prefix)
```

### The Solution: Character Streams

**Idea**: Create a separate abstraction for **character data** (text) vs. **byte data** (binary).

```java
class Writer {
    private OutputStream out;
    private String encoding = "UTF-8";

    void write(char c) {
        byte[] bytes = String.valueOf(c).getBytes(encoding);
        out.write(bytes);
    }

    void write(String text) {
        byte[] bytes = text.getBytes(encoding);
        out.write(bytes);
    }
}
```

### Using It

```java
Writer writer = new Writer();

writer.write("Hello");        // Just writes "Hello", no length prefix
writer.write('\n');            // Write a newline character
writer.write("World");
```

**File contents**:
```
Hello
World
```

Perfect! A readable text file.

### The Encoding Problem

**Plot twist**: How do we represent characters as bytes?

- ASCII: 1 byte per character (only 128 characters)
- UTF-8: 1-4 bytes per character (all of Unicode)
- UTF-16: 2 bytes per character (Java's internal format)

**Character streams handle encoding automatically!**

```java
Writer writer = new Writer("UTF-8");
writer.write("Hello 世界");  // Mixed English + Chinese
// Correct UTF-8 encoding happens automatically
```

### Real Java API

```java
FileOutputStream fos = new FileOutputStream("output.txt");
OutputStreamWriter osw = new OutputStreamWriter(fos, "UTF-8");

osw.write("Hello World\n");
osw.write("你好世界\n");

osw.close();
```

**Why `OutputStreamWriter`?** It **bridges** byte streams and character streams!

---

## Chapter 7: The Convenience Crisis

### The Problem

Writing line by line is tedious:

```java
writer.write("Line 1");
writer.write('\n');
writer.write("Line 2");
writer.write('\n');
```

**We want**:
```java
writer.writeLine("Line 1");
writer.writeLine("Line 2");
```

### The Solution: PrintWriter

```java
class PrintWriter {
    private Writer out;

    void println(String text) {
        out.write(text);
        out.write(System.lineSeparator());  // Cross-platform newline
    }

    void printf(String format, Object... args) {
        String formatted = String.format(format, args);
        out.write(formatted);
    }
}
```

### Using It

```java
PrintWriter pw = new PrintWriter(new FileWriter("output.txt"));

pw.println("Hello World");
pw.printf("Score: %d, Name: %s%n", 9000, "Alice");

pw.close();
```

**Real Java API**:
```java
try (PrintWriter pw = new PrintWriter("output.txt")) {
    pw.println("Hello World");
    pw.printf("Score: %d%n", 9000);
}
```

**Why `PrintWriter`?** High-level, convenient text output!

---

## Chapter 8: The Reading Side

### The Problem

We've been writing files. Now we need to **read** them back!

### Attempt 1: InputStream

Mirror of `OutputStream`:

```java
class InputStream {
    private long currentPosition = 0;

    int read() {
        byte b = readByteFromDisk(currentPosition);
        currentPosition++;
        return b & 0xFF;  // Return as int (0-255), or -1 for EOF
    }

    int read(byte[] buffer) {
        int count = 0;
        for (int i = 0; i < buffer.length; i++) {
            int b = read();
            if (b == -1) break;
            buffer[i] = (byte)b;
            count++;
        }
        return count;
    }
}
```

### Attempt 2: DataInputStream

Read typed data:

```java
class DataInputStream {
    private InputStream in;

    int readInt() {
        int b1 = in.read();
        int b2 = in.read();
        int b3 = in.read();
        int b4 = in.read();
        return (b1 << 24) | (b2 << 16) | (b3 << 8) | b4;
    }

    String readUTF() {
        int length = readInt();
        byte[] bytes = new byte[length];
        in.read(bytes);
        return new String(bytes, "UTF-8");
    }
}
```

### Attempt 3: BufferedInputStream

Just like `BufferedOutputStream`, but for reading:

```java
class BufferedInputStream {
    private InputStream in;
    private byte[] buffer = new byte[8192];
    private int position = 0;
    private int limit = 0;

    int read() {
        if (position >= limit) {
            // Refill buffer
            limit = in.read(buffer);
            position = 0;
            if (limit == -1) return -1;  // EOF
        }
        return buffer[position++] & 0xFF;
    }
}
```

**Performance benefit**: Same as writing — bulk reads instead of one byte at a time.

### Attempt 4: Reader & BufferedReader

For reading text:

```java
class Reader {
    private InputStream in;
    private String encoding = "UTF-8";

    int read() {
        // Read bytes and decode to char
        // (Simplified; real implementation handles multi-byte characters)
    }
}

class BufferedReader {
    private Reader in;
    private char[] buffer = new char[8192];

    String readLine() {
        StringBuilder line = new StringBuilder();
        int c;
        while ((c = in.read()) != -1) {
            if (c == '\n') break;
            if (c == '\r') {
                // Handle \r\n (Windows) vs \n (Unix)
                int next = in.read();
                if (next != '\n') {
                    // Put it back (complicated!)
                }
                break;
            }
            line.append((char)c);
        }
        return line.length() > 0 ? line.toString() : null;
    }
}
```

### Real Usage

```java
try (BufferedReader br = new BufferedReader(new FileReader("input.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}
```

**Why `BufferedReader`?** Efficient line-by-line text reading!

---

## Chapter 9: The Class Explosion Problem

### The Situation

After all our inventions, we have:

**Byte Streams**:
- `OutputStream` / `InputStream`
- `FileOutputStream` / `FileInputStream`
- `BufferedOutputStream` / `BufferedInputStream`
- `DataOutputStream` / `DataInputStream`

**Character Streams**:
- `Writer` / `Reader`
- `FileWriter` / `FileReader`
- `BufferedWriter` / `BufferedReader`
- `PrintWriter`
- `OutputStreamWriter` / `InputStreamReader` (bridges)

**Your colleague**: "This is confusing! Why so many classes?"

### The Design Pattern: Decorator

Each class has a **single responsibility**:

```
FileOutputStream        → Raw file access (bytes)
BufferedOutputStream    → Adds buffering (performance)
DataOutputStream        → Adds typed writing (convenience)
```

**Compose them**:

```java
// Raw file write
FileOutputStream fos = new FileOutputStream("data.bin");

// Add buffering
BufferedOutputStream bos = new BufferedOutputStream(fos);

// Add typed writing
DataOutputStream dos = new DataOutputStream(bos);

// Now you have: typed, buffered, file writing!
dos.writeInt(42);
```

**This is the Decorator pattern!**

### Why This Design?

**Flexibility**:
```java
// Write to file
DataOutputStream dos1 = new DataOutputStream(
    new BufferedOutputStream(
        new FileOutputStream("file.bin")
    )
);

// Write to network socket
DataOutputStream dos2 = new DataOutputStream(
    new BufferedOutputStream(
        socket.getOutputStream()
    )
);

// Write to memory
DataOutputStream dos3 = new DataOutputStream(
    new ByteArrayOutputStream()
);
```

**Same `DataOutputStream` code works for file, network, or memory!**

### The Tradeoff

**Pro**: Flexible, composable, follows Single Responsibility Principle

**Con**: Verbose, confusing for beginners

**Java's choice**: Favor flexibility.

---

## Chapter 10: The Modern Era — NIO & Files

### The Problem (Year 2000)

The old I/O (`java.io`) has issues:

1. **Blocking**: `read()` blocks the thread until data arrives
2. **No metadata**: Hard to check if file exists, get size, etc.
3. **Poor error handling**: `File.delete()` returns `boolean`, not why it failed
4. **No memory-mapped I/O**: Can't map files directly to memory for speed

### The Solution: NIO (New I/O)

Introduced in Java 1.4, refined in Java 7 (NIO.2):

```java
import java.nio.file.*;

// Better file operations
Path path = Paths.get("data.txt");

// Check existence
boolean exists = Files.exists(path);

// Get metadata
long size = Files.size(path);
FileTime modified = Files.getLastModifiedTime(path);

// Read entire file (convenience!)
List<String> lines = Files.readAllLines(path);

// Write entire file
Files.write(path, lines);
```

### Why NIO?

**Convenience methods**:
```java
// Old way (java.io)
BufferedReader br = new BufferedReader(new FileReader("data.txt"));
List<String> lines = new ArrayList<>();
String line;
while ((line = br.readLine()) != null) {
    lines.add(line);
}
br.close();

// New way (java.nio)
List<String> lines = Files.readAllLines(Paths.get("data.txt"));
```

**Better performance (for some use cases)**:
```java
// Memory-mapped file I/O (ultra-fast for large files)
FileChannel channel = FileChannel.open(path, StandardOpenOption.READ);
MappedByteBuffer buffer = channel.map(FileChannel.MapMode.READ_ONLY, 0, channel.size());

// Now you can access file contents like an array in memory!
byte b = buffer.get(1000);  // Instant access, no disk read!
```

**Non-blocking I/O**:
```java
// For network servers handling thousands of connections
Selector selector = Selector.open();
SocketChannel socket = SocketChannel.open();
socket.configureBlocking(false);
socket.register(selector, SelectionKey.OP_READ);

// One thread handles many connections!
```

### When to Use What?

**Use `java.io` when**:
- Reading/writing text line-by-line
- Streaming large files incrementally
- Working with streams (network, etc.)

**Use `java.nio` when**:
- Simple file operations (read all, write all)
- Need file metadata
- High-performance I/O (memory-mapped files)
- Non-blocking network I/O

---

## Chapter 11: The Java File API Family Tree

### The Complete Picture

```
═══════════════════════════════════════════════════════════════
                    JAVA FILE I/O EVOLUTION
═══════════════════════════════════════════════════════════════

LAYER 1: Raw Byte Access
┌────────────────────────────────────────────────────────────┐
│ OutputStream / InputStream                                 │
│ • write(byte) / read() → int                              │
│ • The foundation: everything builds on this               │
└────────────────────────────────────────────────────────────┘

LAYER 2: File Access
┌────────────────────────────────────────────────────────────┐
│ FileOutputStream / FileInputStream                         │
│ • Connects byte streams to actual files on disk           │
│ • Constructor: new FileInputStream("file.txt")            │
└────────────────────────────────────────────────────────────┘

LAYER 3: Performance (Buffering)
┌────────────────────────────────────────────────────────────┐
│ BufferedOutputStream / BufferedInputStream                 │
│ • Adds 8KB buffer to reduce disk writes                   │
│ • 1000x performance improvement                            │
│ • Wraps any OutputStream/InputStream                       │
└────────────────────────────────────────────────────────────┘

LAYER 4: Typed Data
┌────────────────────────────────────────────────────────────┐
│ DataOutputStream / DataInputStream                         │
│ • writeInt(), writeDouble(), writeUTF()                   │
│ • Handles type serialization automatically                │
│ • Binary format                                            │
└────────────────────────────────────────────────────────────┘

LAYER 5: Character Encoding
┌────────────────────────────────────────────────────────────┐
│ Writer / Reader                                            │
│ • Work with characters, not bytes                         │
│ • Handle encoding (UTF-8, UTF-16, etc.)                   │
│ • FileWriter / FileReader for files                       │
└────────────────────────────────────────────────────────────┘

LAYER 6: Character Performance
┌────────────────────────────────────────────────────────────┐
│ BufferedWriter / BufferedReader                            │
│ • Buffered character I/O                                   │
│ • readLine() / write(String)                              │
│ • The workhorse for text files                            │
└────────────────────────────────────────────────────────────┘

LAYER 7: Convenience
┌────────────────────────────────────────────────────────────┐
│ PrintWriter                                                │
│ • println(), printf()                                     │
│ • Never throws IOException (checks with checkError())     │
│ • Similar to System.out                                    │
└────────────────────────────────────────────────────────────┘

BRIDGE LAYER: Byte ↔ Character
┌────────────────────────────────────────────────────────────┐
│ OutputStreamWriter / InputStreamReader                     │
│ • Converts between byte and character streams             │
│ • Handles encoding/decoding                                │
│ • Example: new OutputStreamWriter(fos, "UTF-8")          │
└────────────────────────────────────────────────────────────┘

MODERN ERA: NIO (java.nio.file)
┌────────────────────────────────────────────────────────────┐
│ Files, Path, FileChannel                                   │
│ • Files.readAllLines(path)                                │
│ • Files.write(path, bytes)                                │
│ • Memory-mapped files                                      │
│ • Non-blocking I/O                                         │
│ • Better error handling                                    │
└────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════
```

---

## Chapter 12: Common Patterns — Real-World Usage

### Pattern 1: Reading Text File Line-by-Line

**The Old-School Way** (still common):
```java
try (BufferedReader br = new BufferedReader(new FileReader("input.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}
```

**The Modern Way**:
```java
Files.lines(Paths.get("input.txt"))
     .forEach(System.out::println);
```

### Pattern 2: Writing Text File

**Old-School**:
```java
try (PrintWriter pw = new PrintWriter(new FileWriter("output.txt"))) {
    pw.println("Line 1");
    pw.println("Line 2");
}
```

**Modern**:
```java
List<String> lines = Arrays.asList("Line 1", "Line 2");
Files.write(Paths.get("output.txt"), lines);
```

### Pattern 3: Reading Binary Data

```java
try (DataInputStream dis = new DataInputStream(
        new BufferedInputStream(
            new FileInputStream("data.bin")))) {

    int id = dis.readInt();
    String name = dis.readUTF();
    double score = dis.readDouble();
}
```

### Pattern 4: Copying Files

**Manual (educational)**:
```java
try (InputStream in = new FileInputStream("source.bin");
     OutputStream out = new FileOutputStream("dest.bin")) {

    byte[] buffer = new byte[8192];
    int bytesRead;
    while ((bytesRead = in.read(buffer)) != -1) {
        out.write(buffer, 0, bytesRead);
    }
}
```

**Modern (use this)**:
```java
Files.copy(Paths.get("source.bin"), Paths.get("dest.bin"),
           StandardCopyOption.REPLACE_EXISTING);
```

### Pattern 5: Reading Entire File

**Small text file**:
```java
String content = new String(Files.readAllBytes(Paths.get("file.txt")), "UTF-8");
// OR
List<String> lines = Files.readAllLines(Paths.get("file.txt"));
```

**Large file (streaming)**:
```java
try (Stream<String> stream = Files.lines(Paths.get("large.txt"))) {
    stream.filter(line -> line.contains("error"))
          .forEach(System.out::println);
}
```

---

## Chapter 13: The Encoding Trap

### The Silent Killer

**Problem**: You write a file on Windows, send it to Linux, and... corruption!

```java
// On Windows (default encoding: Windows-1252)
PrintWriter pw = new PrintWriter("data.txt");
pw.println("Café");  // é = byte 0xE9 in Windows-1252
pw.close();

// On Linux (default encoding: UTF-8)
BufferedReader br = new BufferedReader(new FileReader("data.txt"));
String line = br.readLine();
System.out.println(line);  // Output: "Caf�" (corruption!)
```

### The Root Cause

**Java uses platform default encoding when you don't specify!**

- Windows: Windows-1252 (or others)
- Linux: UTF-8
- Mac: UTF-8

**Different bytes for the same character!**

### The Solution: Always Specify Encoding

**Wrong**:
```java
FileWriter fw = new FileWriter("data.txt");  // Uses default encoding!
```

**Right**:
```java
OutputStreamWriter osw = new OutputStreamWriter(
    new FileOutputStream("data.txt"),
    StandardCharsets.UTF_8  // Explicit encoding!
);
```

**Modern NIO**:
```java
Files.write(Paths.get("data.txt"),
            "Café".getBytes(StandardCharsets.UTF_8));

// OR
Files.writeString(Paths.get("data.txt"), "Café", StandardCharsets.UTF_8);
```

### Best Practice

**Always use UTF-8 unless you have a specific reason not to.**

```java
import java.nio.charset.StandardCharsets;

// Reading
BufferedReader br = new BufferedReader(
    new InputStreamReader(
        new FileInputStream("file.txt"),
        StandardCharsets.UTF_8
    )
);

// Writing
BufferedWriter bw = new BufferedWriter(
    new OutputStreamWriter(
        new FileOutputStream("file.txt"),
        StandardCharsets.UTF_8
    )
);
```

---

## Chapter 14: The Resource Leak Problem

### The Bug

```java
FileInputStream fis = new FileInputStream("data.bin");
int data = fis.read();

// ... some code ...

if (error) {
    return;  // BUG: Stream never closed!
}

fis.close();  // This line might never execute
```

**Consequence**:
- File handle leaked
- File remains locked (can't delete/rename)
- Run out of file descriptors (OS limit)

### The Old Solution: try-finally

```java
FileInputStream fis = null;
try {
    fis = new FileInputStream("data.bin");
    int data = fis.read();
    // ... process data ...
} finally {
    if (fis != null) {
        fis.close();  // Always executes
    }
}
```

**Verbose but safe.**

### The Modern Solution: try-with-resources

```java
try (FileInputStream fis = new FileInputStream("data.bin")) {
    int data = fis.read();
    // ... process data ...
}  // fis.close() called automatically!
```

**How it works**: Any class implementing `AutoCloseable` can be used in try-with-resources.

**Multiple resources**:
```java
try (FileInputStream in = new FileInputStream("input.bin");
     FileOutputStream out = new FileOutputStream("output.bin")) {

    // Copy data
    byte[] buffer = new byte[8192];
    int bytesRead;
    while ((bytesRead = in.read(buffer)) != -1) {
        out.write(buffer, 0, bytesRead);
    }
}  // Both streams closed automatically, in reverse order
```

**Best Practice**: Always use try-with-resources for I/O!

---

## Chapter 15: Performance Deep Dive

### Experiment: Writing 1 Million Integers

Let's measure different approaches:

#### Approach 1: Unbuffered FileOutputStream
```java
long start = System.nanoTime();
try (FileOutputStream fos = new FileOutputStream("data.bin")) {
    for (int i = 0; i < 1_000_000; i++) {
        fos.write(i >>> 24);
        fos.write(i >>> 16);
        fos.write(i >>> 8);
        fos.write(i);
    }
}
long time = (System.nanoTime() - start) / 1_000_000;  // Convert to ms
System.out.println("Time: " + time + " ms");
```
**Result**: ~45,000 ms (45 seconds!) 🐌

#### Approach 2: BufferedOutputStream
```java
try (BufferedOutputStream bos = new BufferedOutputStream(
        new FileOutputStream("data.bin"))) {
    for (int i = 0; i < 1_000_000; i++) {
        bos.write(i >>> 24);
        bos.write(i >>> 16);
        bos.write(i >>> 8);
        bos.write(i);
    }
}
```
**Result**: ~120 ms (375x faster!) 🚀

#### Approach 3: DataOutputStream + BufferedOutputStream
```java
try (DataOutputStream dos = new DataOutputStream(
        new BufferedOutputStream(
            new FileOutputStream("data.bin")))) {
    for (int i = 0; i < 1_000_000; i++) {
        dos.writeInt(i);
    }
}
```
**Result**: ~100 ms (cleaner code, similar performance) 🚀

#### Approach 4: ByteBuffer (NIO)
```java
try (FileChannel channel = FileChannel.open(Paths.get("data.bin"),
        StandardOpenOption.WRITE, StandardOpenOption.CREATE)) {

    ByteBuffer buffer = ByteBuffer.allocate(4 * 1_000_000);
    for (int i = 0; i < 1_000_000; i++) {
        buffer.putInt(i);
    }
    buffer.flip();
    channel.write(buffer);
}
```
**Result**: ~40 ms (fastest!) 🚀🚀

### The Lesson

**Buffering is essential!** Unbuffered I/O is 100-1000x slower.

**Why?**
- Unbuffered: 4 million system calls
- Buffered: ~500 system calls (8KB buffer)
- NIO ByteBuffer: 1 system call (bulk write)

---

## Chapter 16: The Philosophy — Why This Design?

### Principle 1: Separation of Concerns

Each class has **one job**:

- `FileInputStream`: Access file
- `BufferedInputStream`: Add buffering
- `DataInputStream`: Handle typed data
- `InputStreamReader`: Convert bytes to characters
- `BufferedReader`: Add character buffering

**Why not one "SuperFileReader"?**

Because you might want:
- Buffered file reading
- Buffered network reading
- Buffered memory reading

**Composability** > **Convenience**

### Principle 2: Streams as Abstractions

A **stream** is a sequence of data. It could come from:
- A file
- A network socket
- Memory
- Generated on-the-fly

**Same interface works everywhere!**

```java
// All take InputStream
void processData(InputStream in) {
    // Doesn't care if it's file, network, or memory
}

processData(new FileInputStream("file.bin"));
processData(socket.getInputStream());
processData(new ByteArrayInputStream(bytes));
```

### Principle 3: Byte Streams vs. Character Streams

**Why two separate hierarchies?**

**Bytes are universal**:
- Binary data (images, audio, compiled code)
- Low-level protocols

**Characters are human**:
- Text files
- Different encodings (UTF-8, UTF-16, etc.)

Mixing them causes encoding bugs. **Separate hierarchies prevent mistakes.**

### Principle 4: Evolution Without Breaking

Java couldn't replace `java.io` with `java.nio` because:
- Millions of lines of existing code
- Different use cases

**Solution**: Add `java.nio`, keep `java.io`. Let developers choose.

**Trade-off**: More APIs to learn, but backward compatibility maintained.

---

## Chapter 17: Common Mistakes & How to Avoid Them

### Mistake 1: Forgetting to Close Streams

**Wrong**:
```java
FileWriter fw = new FileWriter("output.txt");
fw.write("Hello");
// Forgot to close! Data might not be written!
```

**Why bad**: Buffered data not flushed, file handle leaked.

**Fix**: Use try-with-resources.

### Mistake 2: Not Specifying Encoding

**Wrong**:
```java
FileWriter fw = new FileWriter("output.txt");  // Platform-dependent!
```

**Fix**:
```java
OutputStreamWriter osw = new OutputStreamWriter(
    new FileOutputStream("output.txt"),
    StandardCharsets.UTF_8
);
```

### Mistake 3: Reading Large File into Memory

**Wrong**:
```java
// 2GB file → OutOfMemoryError!
byte[] data = Files.readAllBytes(Paths.get("huge.log"));
```

**Fix**: Stream it.
```java
try (Stream<String> lines = Files.lines(Paths.get("huge.log"))) {
    lines.forEach(System.out::println);
}
```

### Mistake 4: Ignoring Exceptions

**Wrong**:
```java
try {
    FileInputStream fis = new FileInputStream("file.txt");
} catch (IOException e) {
    // Silently ignore
}
```

**Fix**: Log it, rethrow it, or handle it properly.

### Mistake 5: Using FileReader Instead of Files

**Outdated**:
```java
BufferedReader br = new BufferedReader(new FileReader("data.txt"));
```

**Modern**:
```java
BufferedReader br = Files.newBufferedReader(Paths.get("data.txt"), StandardCharsets.UTF_8);
```

---

## Chapter 18: Putting It All Together

### The Big Picture

**You now understand**:
1. **Why files exist**: Persistence in a volatile world
2. **Why OutputStreams exist**: Abstraction over raw bytes
3. **Why BufferedOutputStream exists**: Performance (bulk I/O)
4. **Why DataOutputStream exists**: Convenience (typed data)
5. **Why Writers exist**: Character encoding
6. **Why BufferedReader exists**: Efficient text reading
7. **Why PrintWriter exists**: High-level convenience
8. **Why NIO exists**: Modern APIs, better performance

### Decision Tree: Which API to Use?

```
Do you need to read/write a file?
│
├─ Text file?
│  │
│  ├─ Small file (< 100 MB)?
│  │  └─ Use Files.readAllLines() / Files.write()
│  │
│  └─ Large file or streaming?
│     └─ Use BufferedReader / BufferedWriter
│        (with explicit UTF-8 encoding!)
│
└─ Binary file?
   │
   ├─ Simple byte operations?
   │  └─ Use Files.readAllBytes() / Files.write()
   │
   ├─ Typed data (int, double, etc.)?
   │  └─ Use DataInputStream / DataOutputStream
   │     (wrapped with BufferedInputStream/OutputStream)
   │
   └─ High performance / large files?
      └─ Use FileChannel with ByteBuffer (NIO)
```

### The Perfect Read Example

```java
// Reading a text file (small)
List<String> lines = Files.readAllLines(Paths.get("config.txt"), StandardCharsets.UTF_8);

// Reading a text file (large, line by line)
try (BufferedReader br = Files.newBufferedReader(Paths.get("log.txt"), StandardCharsets.UTF_8)) {
    String line;
    while ((line = br.readLine()) != null) {
        processLine(line);
    }
}

// Reading a binary file (typed data)
try (DataInputStream dis = new DataInputStream(
        new BufferedInputStream(
            Files.newInputStream(Paths.get("data.bin"))))) {

    int header = dis.readInt();
    String name = dis.readUTF();
    double value = dis.readDouble();
}
```

### The Perfect Write Example

```java
// Writing a text file (small)
List<String> lines = Arrays.asList("Line 1", "Line 2", "Line 3");
Files.write(Paths.get("output.txt"), lines, StandardCharsets.UTF_8);

// Writing a text file (large, streaming)
try (BufferedWriter bw = Files.newBufferedWriter(Paths.get("output.txt"), StandardCharsets.UTF_8)) {
    for (String line : hugeDataSet) {
        bw.write(line);
        bw.newLine();
    }
}

// Writing binary data (typed)
try (DataOutputStream dos = new DataOutputStream(
        new BufferedOutputStream(
            Files.newOutputStream(Paths.get("data.bin"))))) {

    dos.writeInt(42);
    dos.writeUTF("Alice");
    dos.writeDouble(3.14159);
}
```

---

## Conclusion: You Just Invented Java File I/O

### The Journey

You started with a simple problem: **save data when the computer turns off**.

You invented:
1. **Raw byte writing** (too tedious)
2. **OutputStreams** (better abstraction)
3. **DataOutputStreams** (typed data)
4. **Buffering** (massive performance gain)
5. **Character streams** (encoding support)
6. **High-level APIs** (convenience)
7. **NIO** (modern, powerful)

**You didn't memorize. You *derived* the solution from first principles.**

### The Meta-Lesson

**This is how you should learn any API**:
1. Understand the **problem** it solves
2. Try the **naive solution** (see why it fails)
3. Discover the **incremental improvements**
4. Appreciate the **trade-offs**

**Now when you see**:
```java
new DataOutputStream(new BufferedOutputStream(new FileOutputStream("file.bin")))
```

**You don't think**: "What a mess!"

**You think**: "Ah, they want typed data (`DataOutputStream`), with buffering for performance (`BufferedOutputStream`), writing to a file (`FileOutputStream`). Makes sense!"

### Next Steps

**Practice**:
- Implement a simple key-value store (save/load data)
- Write a log file parser
- Build a config file reader
- Create a binary file format

**Explore**:
- Read the JavaDoc for `java.io` and `java.nio`
- Implement your own `BufferedOutputStream` from scratch
- Study how `Files.lines()` works internally

**Think**:
- What if we had infinite RAM? (Would we need files?)
- What if disks were as fast as RAM? (Would we need buffering?)
- What if all data was text? (Would we need byte streams?)

### Final Thought

**File I/O isn't just about reading and writing data. It's a case study in:**
- Abstraction layers
- Performance optimization
- API design
- Backward compatibility
- Trade-offs and philosophy

**You now understand the "why" behind every Java File class.**

**Go forth and write some I/O code. You've earned it.**

---

## Appendix A: Quick Reference

### Reading Text Files

```java
// Small file
List<String> lines = Files.readAllLines(Paths.get("file.txt"), StandardCharsets.UTF_8);

// Large file (streaming)
try (BufferedReader br = Files.newBufferedReader(Paths.get("file.txt"), StandardCharsets.UTF_8)) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}

// Java 8+ Stream API
Files.lines(Paths.get("file.txt"), StandardCharsets.UTF_8)
     .forEach(System.out::println);
```

### Writing Text Files

```java
// Small file
List<String> lines = Arrays.asList("Line 1", "Line 2");
Files.write(Paths.get("file.txt"), lines, StandardCharsets.UTF_8);

// Large file (streaming)
try (BufferedWriter bw = Files.newBufferedWriter(Paths.get("file.txt"), StandardCharsets.UTF_8)) {
    bw.write("Line 1");
    bw.newLine();
    bw.write("Line 2");
}

// PrintWriter (convenience)
try (PrintWriter pw = new PrintWriter(Files.newBufferedWriter(Paths.get("file.txt"), StandardCharsets.UTF_8))) {
    pw.println("Line 1");
    pw.printf("Score: %d%n", 100);
}
```

### Reading Binary Files

```java
// Small file
byte[] bytes = Files.readAllBytes(Paths.get("file.bin"));

// Typed data
try (DataInputStream dis = new DataInputStream(
        new BufferedInputStream(
            Files.newInputStream(Paths.get("file.bin"))))) {

    int value = dis.readInt();
    String text = dis.readUTF();
}
```

### Writing Binary Files

```java
// Small file
byte[] bytes = {1, 2, 3, 4};
Files.write(Paths.get("file.bin"), bytes);

// Typed data
try (DataOutputStream dos = new DataOutputStream(
        new BufferedOutputStream(
            Files.newOutputStream(Paths.get("file.bin"))))) {

    dos.writeInt(42);
    dos.writeUTF("Hello");
}
```

### File Operations

```java
// Check existence
boolean exists = Files.exists(Paths.get("file.txt"));

// Delete
Files.delete(Paths.get("file.txt"));

// Copy
Files.copy(Paths.get("source.txt"), Paths.get("dest.txt"), StandardCopyOption.REPLACE_EXISTING);

// Move
Files.move(Paths.get("old.txt"), Paths.get("new.txt"), StandardCopyOption.REPLACE_EXISTING);

// Get size
long size = Files.size(Paths.get("file.txt"));

// List directory
try (Stream<Path> paths = Files.list(Paths.get("."))) {
    paths.forEach(System.out::println);
}
```

---

## Appendix B: Performance Comparison Table

| Operation | Time (1M integers) | Relative Speed |
|-----------|-------------------|----------------|
| Unbuffered write | 45,000 ms | 1x (baseline) |
| Buffered write | 120 ms | 375x faster |
| DataOutputStream + Buffer | 100 ms | 450x faster |
| NIO ByteBuffer | 40 ms | 1,125x faster |

**Key Takeaway**: Always use buffering!

---

## Appendix C: Class Hierarchy Cheat Sheet

### Byte Streams
```
OutputStream
├── FileOutputStream
├── ByteArrayOutputStream
└── FilterOutputStream
    ├── BufferedOutputStream
    ├── DataOutputStream
    └── PrintStream

InputStream
├── FileInputStream
├── ByteArrayInputStream
└── FilterInputStream
    ├── BufferedInputStream
    └── DataInputStream
```

### Character Streams
```
Writer
├── OutputStreamWriter
│   └── FileWriter
├── BufferedWriter
├── PrintWriter
└── StringWriter

Reader
├── InputStreamReader
│   └── FileReader
├── BufferedReader
└── StringReader
```

### NIO
```
java.nio.file.Files
java.nio.file.Path
java.nio.channels.FileChannel
java.nio.ByteBuffer
```

---

**Version**: 1.0
**Date**: November 2025
**License**: Creative Commons BY-SA 4.0

---

*"The best way to understand an API is to imagine inventing it yourself."*

**Thank you for taking this journey. Now you don't just use File I/O—you understand it.**
