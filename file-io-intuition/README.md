# Inventing File I/O from First Principles

## Building Intuition for File Operations

### What is this?

A mini-book that teaches file I/O by **inventing it from scratch**. Instead of memorizing Java's `FileInputStream`, `BufferedReader`, and dozens of other classes, you'll understand **why they exist** by deriving them from first principles.

### Who is this for?

**Software developers and students** who:
- Have used Java's File classes but never understood why there are so many
- Want to understand the "why" behind `BufferedOutputStream`, `DataInputStream`, etc.
- Are tired of cargo-culting `new BufferedReader(new InputStreamReader(...))` without understanding it
- Want to build deep intuition about I/O performance

### What you'll learn

Starting from "we need to save data when the computer turns off," you'll discover:

1. **Why OutputStreams exist**: Abstracting raw byte access
2. **Why Buffering exists**: Performance (1000x speed improvement!)
3. **Why DataStreams exist**: Typed data convenience
4. **Why Writers/Readers exist**: Character encoding
5. **Why there are so many classes**: Decorator pattern and composition
6. **Why NIO exists**: Modern APIs and performance
7. **When to use what**: Practical decision trees

### How is this different?

**Traditional approach**: "Here's `FileInputStream`. It reads bytes from a file. Memorize the API."

**This book's approach**:
1. Start with the problem (volatile RAM)
2. Try the naive solution (write bytes manually)
3. Hit performance wall (1000x too slow!)
4. Invent buffering (huge win!)
5. Add convenience layers (typed data, character encoding)
6. Understand trade-offs (verbosity vs. flexibility)

**Result**: Deep understanding instead of shallow memorization.

### Read the book

👉 **[Start reading: book.md](book.md)**

### What makes this stick in your mind?

- **First-principles thinking**: Derive solutions instead of being told
- **Visual hierarchy**: Clear structure for easy scanning
- **Performance comparisons**: See why buffering matters (45 seconds → 0.1 seconds!)
- **Code examples**: Concrete demonstrations at every step
- **Common mistakes**: Learn what to avoid
- **Decision trees**: Know which API to use when

### Philosophy

> "The best way to understand an API is to imagine inventing it yourself."

This book is part of a series exploring computing concepts from first principles. Check out the other books in this repository!

---

**Version**: 1.0
**License**: Creative Commons BY-SA 4.0
