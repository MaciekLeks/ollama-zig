# 🦙⚡ ollama-zig

The Ollama zig library is the easiest way to interact and integrate your Zig project with the [Ollama REST API](https://github.com/ollama/ollama/blob/main/docs/api.md).

## Features

- ✅ Chat API (with streaming support)
- ✅ Generate API (with streaming support)
- ✅ List models
- ✅ Show model information
- ✅ Generate embeddings
- ✅ Full type safety with Zig's compile-time checks
- ✅ Zero runtime dependencies
- ✅ Streaming response support through iterator interface

## Requirements

- **Zig 0.15.1+** (uses the new `std.Io.Reader`/`std.Io.Writer` API)
- [Ollama](https://ollama.ai/) running locally or remotely

## Installation

### Method 1: Using `zig fetch`

1. Clone or download this repository, then run:

   ```bash
   zig fetch --save path/to/ollama-zig
   ```

   Or use the GitHub URL directly:

   ```bash
   zig fetch --save https://github.com/naamfung/ollama-zig/archive/main.tar.gz
   ```

2. Add the dependency and module to your `build.zig`:

   ```zig
   const std = @import("std");

   pub fn build(b: *std.Build) void {
       const target = b.standardTargetOptions(.{});
       const optimize = b.standardOptimizeOption(.{});

       // Add ollama-zig dependency
       const ollama_dep = b.dependency("ollama-zig", .{
           .target = target,
           .optimize = optimize,
       });
       const ollama_mod = ollama_dep.module("ollama");

       // Create your executable module
       const exe_mod = b.createModule(.{
           .root_source_file = b.path("src/main.zig"),
           .target = target,
           .optimize = optimize,
       });
       exe_mod.addImport("ollama", ollama_mod);

       const exe = b.addExecutable(.{
           .name = "my-app",
           .root_module = exe_mod,
       });
       b.installArtifact(exe);
   }
   ```

3. Import it inside your project:

   ```zig
   const Ollama = @import("ollama");
   ```

### Method 2: Copy source files

Simply copy `src/ollama.zig` and `src/types.zig` to your project and import them directly.

## Quick Start

### Basic Chat

```zig
const std = @import("std");
const Ollama = @import("ollama");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Initialize client
    var ollama = Ollama.init(allocator, .{});
    defer ollama.deinit();

    // Prepare messages
    var msgs = try std.ArrayList(Ollama.Type.Message).initCapacity(allocator, 1);
    defer msgs.deinit(allocator);
    try msgs.append(allocator, .{ .role = "user", .content = "Why is the sky blue?" });

    // Send chat request (non-streaming)
    const response = try ollama.chat(.{
        .model = "llama3.2",
        .messages = msgs.items,
        .stream = false,
    });
    defer response.deinit();

    std.debug.print("{s}\n", .{response.value.message.content});
}
```

### Streaming Chat

```zig
const std = @import("std");
const Ollama = @import("ollama");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var ollama = Ollama.init(allocator, .{});
    defer ollama.deinit();

    var msgs = try std.ArrayList(Ollama.Type.Message).initCapacity(allocator, 1);
    defer msgs.deinit(allocator);
    try msgs.append(allocator, .{ .role = "user", .content = "Write a haiku about coding" });

    // Streaming returns an iterator
    var stream = try ollama.chatStream(.{
        .model = "llama3.2",
        .messages = msgs.items,
    });

    // Iterate through response chunks
    while (try stream.next()) |part| {
        std.debug.print("{s}", .{part.message.content});
    }
}
```

### Generate Response

```zig
const std = @import("std");
const Ollama = @import("ollama");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var ollama = Ollama.init(allocator, .{});
    defer ollama.deinit();

    // Non-streaming generate
    const response = try ollama.generate(.{
        .model = "llama3.2",
        .prompt = "Why is the sky blue?",
        .stream = false,
    });
    defer response.deinit();

    std.debug.print("{s}\n", .{response.value.response});
}
```

## API Reference

### Configuration

```zig
const config = Ollama.Type.Config{
    .host = "http://127.0.0.1:11434",  // Ollama server URL
    .response_max_size = 1024 * 100,    // Max response size in bytes (default: 4096)
};

var ollama = Ollama.init(allocator, config);
defer ollama.deinit();
```

### Chat API

#### `chat(request: ChatRequest) !Parsed(ChatResponse)`

Generate a single chat response. Set `stream = false`.

```zig
const response = try ollama.chat(.{
    .model = "llama3.2",
    .messages = msgs.items,
    .stream = false,
    .format = "json",                    // Optional: force JSON output
    .options = .{ .temperature = 0.7 },  // Optional: model options
});
```

#### `chatStream(request: ChatRequest) !Streamable(ChatResponse)`

Stream chat responses as an iterator.

```zig
var stream = try ollama.chatStream(.{
    .model = "llama3.2",
    .messages = msgs.items,
});
while (try stream.next()) |part| {
    std.debug.print("{s}", .{part.message.content});
}
```

**ChatRequest fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | `[]const u8` | Yes | Model name (e.g., "llama3.2", "mistral") |
| `messages` | `[]Message` | Yes | Conversation history |
| `stream` | `bool` | No | Enable streaming (default: true) |
| `format` | `?[]const u8` | No | Output format ("json") |
| `options` | `?Options` | No | Model parameters |
| `template` | `?[]const u8` | No | Override prompt template |
| `keep_alive` | `?KeepAlive` | No | Model memory duration |

### Generate API

#### `generate(request: GenerateRequest) !Parsed(GenerateResponse)`

Generate a response for a prompt. Set `stream = false`.

```zig
const response = try ollama.generate(.{
    .model = "llama3.2",
    .prompt = "Explain quantum computing",
    .stream = false,
    .system = "You are a helpful physics teacher.",  // Optional
});
```

#### `generateStream(request: GenerateRequest) !Streamable(GenerateResponse)`

Stream generate responses as an iterator.

**GenerateRequest fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | `[]const u8` | Yes | Model name |
| `prompt` | `[]const u8` | Yes | Input prompt |
| `stream` | `bool` | No | Enable streaming (default: true) |
| `system` | `?[]const u8` | No | System prompt |
| `context` | `?[]const u32` | No | Context from previous response |
| `raw` | `?bool` | No | Skip prompt formatting |

### List Models

```zig
const response = try ollama.list();
defer response.deinit();

for (response.value.models) |model| {
    std.debug.print("Model: {s}, Size: {d} bytes\n", .{ model.name, model.size });
}
```

### Show Model Information

```zig
const response = try ollama.show(.{
    .model = "llama3.2",
});
defer response.deinit();

std.debug.print("Modelfile:\n{s}\n", .{response.value.modelfile});
std.debug.print("Template: {s}\n", .{response.value.template});
std.debug.print("Parameters: {s}\n", .{response.value.parameters});
```

### Generate Embeddings

```zig
const response = try ollama.embeddings(.{
    .model = "llama3.2",
    .prompt = "The quick brown fox jumps over the lazy dog",
});
defer response.deinit();

// Embedding vector
for (response.value.embedding) |value| {
    std.debug.print("{d} ", .{value});
}
```

### Model Options

The `Options` struct provides fine-grained control over model behavior:

```zig
const options = Ollama.Type.Options{
    // Load-time options
    .num_ctx = 4096,           // Context window size
    .num_batch = 512,          // Batch size
    .num_gpu = 1,              // Number of GPU layers
    .use_mmap = true,          // Use memory mapping
    .use_mlock = false,        // Lock memory

    // Runtime options
    .temperature = 0.7,        // Sampling temperature
    .top_p = 0.9,              // Nucleus sampling
    .top_k = 40,               // Top-k sampling
    .seed = 42,                // Random seed
    .num_predict = 256,        // Max tokens to predict
    .stop = &.{ "###", "END" }, // Stop sequences
    .repeat_penalty = 1.1,     // Repetition penalty
};
```

### KeepAlive Union

Control how long models stay loaded in memory:

```zig
// Using string duration
.keep_alive = .{ .string = "5m" }   // 5 minutes

// Using number (milliseconds)
.keep_alive = .{ .number = 300000 } // 5 minutes in ms
```

### Message Type

```zig
const Message = Ollama.Type.Message{
    .role = "user",      // "system", "user", "assistant", or "tool"
    .content = "Hello!",
};
```

## Error Handling

```zig
const response = ollama.chat(.{
    .model = "llama3.2",
    .messages = msgs.items,
    .stream = false,
}) catch |err| {
    switch (err) {
        error.HTTPRequestFailed => {
            std.debug.print("Request failed. Is Ollama running?\n", .{});
        },
        error.StreamNotDisabled => {
            std.debug.print("Set stream = false for non-streaming calls\n", .{});
        },
        else => {
            std.debug.print("Error: {any}\n", .{err});
        },
    }
    return;
};
```

## Complete Example: Conversational Chatbot

```zig
const std = @import("std");
const Ollama = @import("ollama");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var ollama = Ollama.init(allocator, .{});
    defer ollama.deinit();

    // Conversation history
    var history = std.ArrayList(Ollama.Type.Message).initCapacity(allocator, 10) catch return;
    defer history.deinit(allocator);

    // Add system message
    history.append(allocator, .{
        .role = "system",
        .content = "You are a helpful assistant. Be concise.",
    }) catch return;

    const stdin = std.io.getStdIn();
    const stdout = std.io.getStdOut();

    while (true) {
        try stdout.writeAll("\nYou: ");

        var input = std.ArrayList(u8).initCapacity(allocator, 256) catch return;
        defer input.deinit(allocator);

        stdin.reader().streamUntilDelimiter(input.writer(allocator), '\n', null) catch |err| {
            if (err == error.EndOfStream) break;
            return err;
        };

        if (input.items.len == 0) continue;

        // Add user message
        try history.append(allocator, .{
            .role = "user",
            .content = input.items,
        });

        // Stream response
        try stdout.writeAll("Assistant: ");
        var stream = try ollama.chatStream(.{
            .model = "llama3.2",
            .messages = history.items,
        });

        var assistant_response = std.ArrayList(u8).initCapacity(allocator, 1024) catch return;
        defer assistant_response.deinit(allocator);

        while (try stream.next()) |part| {
            try stdout.writeAll(part.message.content);
            try assistant_response.appendSlice(allocator, part.message.content);
        }

        // Add assistant response to history
        try history.append(allocator, .{
            .role = "assistant",
            .content = assistant_response.items,
        });
    }
}
```

## Multi-turn Conversation with Context

For the `generate` API, you can maintain conversation context:

```zig
var ollama = Ollama.init(allocator, .{});
defer ollama.deinit();

// First message
var response = try ollama.generate(.{
    .model = "llama3.2",
    .prompt = "What is the capital of France?",
    .stream = false,
});
defer response.deinit();

// Save context for next turn
const context = response.value.context;

// Continue conversation with context
var response2 = try ollama.generate(.{
    .model = "llama3.2",
    .prompt = "What is its population?",
    .context = context,  // Pass previous context
    .stream = false,
});
defer response2.deinit();
```

## API Endpoints Not Yet Implemented

- `pull` - Pull a model from the registry
- `push` - Push a model to the registry
- `create` - Create a new model
- `delete` - Delete a model
- `copy` - Copy a model

Contributions are welcome!

## Known Issues

- Images in messages are not supported yet
- May return unknown errors if the model is not found on the host system
- The `name` field from `list()` might be empty in some cases

## License

MIT License. See [LICENSE](LICENSE) for full license text.

## Acknowledgments

- [Ollama](https://github.com/ollama/ollama) - The amazing local LLM runtime
- Zig language team for the excellent standard library
