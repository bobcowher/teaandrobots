---
title: "Working with Code Blocks in Hugo"
date: 2025-08-12
tags: ["hugo", "markdown", "code"]
draft: true 
---

This post demonstrates how to properly format code blocks in Hugo with syntax highlighting.

## Python Example

Here's a simple Python function with proper syntax highlighting:

```python
def fibonacci(n):
    """Generate Fibonacci sequence up to n terms."""
    if n <= 0:
        return []
    elif n == 1:
        return [0]
    elif n == 2:
        return [0, 1]
    
    sequence = [0, 1]
    for i in range(2, n):
        sequence.append(sequence[i-1] + sequence[i-2])
    
    return sequence

# Example usage
numbers = fibonacci(10)
print(f"First 10 Fibonacci numbers: {numbers}")
```

## JavaScript Example

Here's a JavaScript example showing async/await patterns:

```javascript
async function fetchUserData(userId) {
    try {
        const response = await fetch(`/api/users/${userId}`);
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const userData = await response.json();
        return userData;
    } catch (error) {
        console.error('Error fetching user data:', error);
        return null;
    }
}

// Usage
fetchUserData(123).then(user => {
    if (user) {
        console.log('User data:', user);
    }
});
```

## Shell/Bash Example

Command line examples are also important:

```bash
# Install dependencies
npm install

# Build the project
npm run build

# Start development server
npm run dev

# Run tests with coverage
npm test -- --coverage
```

## Configuration Example (TOML)

Hugo configuration example:

```toml
baseURL = 'https://example.com'
languageCode = 'en-us'
title = 'My Hugo Site'
theme = "hello-friend"

[markup]
  [markup.goldmark]
    [markup.goldmark.renderer]
      unsafe = true
  [markup.highlight]
    style = "github"
    lineNos = true
```

## Inline Code

You can also use `inline code` within sentences using single backticks.

## Code Block Features

Hugo's built-in syntax highlighting supports:
- Line numbers
- Multiple language support
- Copy-to-clipboard functionality (theme dependent)
- Custom styling through CSS

The hello-friend theme you're using should automatically handle syntax highlighting for these code blocks!
