---
title: I Tried Xcode 26.3's Agentic Coding for a Week. Here's What Happened.
slug: xcode-agentic-coding-tutorial
date: 2026-02-06
author: Sam Okafor
section: tutorials
type: tutorial
---

Alright, let's get into it.

Apple dropped Xcode 26.3 on Monday with Claude Agent and Codex integration. I cleared my calendar, made too much coffee, and spent the week building a side project entirely with AI agents. The results were... educational.

## Setting Up (The Part Where I Nearly Gave Up)

First, you need Xcode 26.3. If you're on macOS 14 or earlier, stop reading and update. The agentic features require Sequoia 15.4 minimum.

```bash
# Check your macOS version
sw_vers

# Should show: ProductVersion: 15.4 or higher
```

Open Xcode, go to Settings → Agentic Coding → Enable. You'll need to authenticate with either Anthropic or OpenAI. I started with Claude Agent because I already had API credits.

```
Settings → Agentic Coding → Providers → Claude Agent → Add API Key
```

This is the part where I messed up: I entered my Anthropic console password instead of the API key. Spent 15 minutes wondering why authentication kept failing. The error message said "Invalid credentials" which could mean anything. Look for the key that starts with `sk-ant-`.

## Your First Agent Task

Once authenticated, you'll see a new panel in the editor sidebar: Agent Tasks. Think of it like a very smart intern sitting next to you.

Create a new Swift file and type a comment:

```swift
// TODO: Create a view that displays a list of tasks with checkboxes
// Each task should have a title, due date, and completion status
// Support swipe-to-delete
```

Right-click the comment, select "Assign to Agent" → "Claude Agent". 

What happens next is legitimately impressive. The agent reads your comment, examines your project structure, checks your dependencies, and starts writing code. You can watch it think in real-time. It took about 40 seconds to produce this:

```swift
import SwiftUI

struct Task: Identifiable {
    let id = UUID()
    var title: String
    var dueDate: Date
    var isCompleted: Bool
}

struct TaskListView: View {
    @State private var tasks: [Task] = []
    
    var body: some View {
        List {
            ForEach($tasks) { $task in
                TaskRow(task: $task)
            }
            .onDelete(perform: deleteTasks)
        }
        .navigationTitle("Tasks")
    }
    
    private func deleteTasks(at offsets: IndexSet) {
        tasks.remove(atOffsets: offsets)
    }
}

struct TaskRow: View {
    @Binding var task: Task
    
    var body: some View {
        HStack {
            Button(action: { task.isCompleted.toggle() }) {
                Image(systemName: task.isCompleted ? "checkmark.circle.fill" : "circle")
            }
            
            VStack(alignment: .leading) {
                Text(task.title)
                    .strikethrough(task.isCompleted)
                Text(task.dueDate, style: .date)
                    .font(.caption)
                    .foregroundColor(.secondary)
            }
        }
    }
}
```

It works. It compiles. The swipe-to-delete is correctly implemented. I would have written this myself in maybe 10 minutes, but seeing it appear line by line while the agent explains its reasoning is a different experience.

## The "Agentic" Part

Here's where it gets interesting. The agent doesn't just write code; it can run builds, read error messages, and fix issues.

I intentionally introduced a bug:

```swift
// Changed Task to be non-Identifiable
struct Task {
    var title: String
    // ... rest of struct, missing Identifiable
}
```

Build failed with a compiler error. I selected the error and clicked "Send to Agent."

The agent read the error, understood the context, added `Identifiable` conformance AND the missing `id` property, then triggered a rebuild. All without me touching the keyboard.

This is where I actually said "oh" out loud to an empty room.

## The Reality Check

Week two, I tried something ambitious: refactoring an existing 2,000-line view controller into SwiftUI with proper MVVM architecture.

The agent started strong. It identified the data flow, suggested a ViewModel structure, began extracting views. About 40% through, things got weird.

It renamed a variable `userProfile` to `profileData` in one file but missed references in three others. Build broke. When I asked it to fix the errors, it introduced new variable names that broke additional references.

This is the part where I messed up (again): I let the agent run too long without reviewing intermediate steps. The lesson is that agents work best in short iterations. Write a feature, review, approve, continue. Not "refactor my entire codebase while I get lunch."

## When to Use Each Agent

Both Claude Agent and Codex are available. After a week, here's my take:

**Claude Agent**: Better at understanding context, reading documentation, explaining why it made choices. Slower, but more careful. Use for new feature implementation and debugging complex issues.

**Codex**: Faster, more comfortable with pure code transformations. Less explanation, more execution. Use for refactoring, test generation, and "just make this work" tasks.

You can switch between them mid-task. Sometimes I start with Claude for the architecture, then switch to Codex for the boilerplate.

## The Commands That Actually Help

```swift
// Agent: implement the NetworkManager class following the protocol defined in NetworkProtocol.swift
```

```swift
// Agent: add unit tests for the calculateTotal function, including edge cases for empty arrays and negative numbers
```

```swift
// Agent: this code has a memory leak, find and fix it
```

That last one is magic. Point it at a Instruments memory graph and watch it work.

## My Honest Take After Seven Days

Agentic coding in Xcode 26.3 is good enough to use every day. It's not good enough to use without supervision. 

I wrote about 60% less code this week. I fixed bugs about 40% faster. I spent about 20% of my time correcting agent mistakes.

The math works out positive, but it's a different kind of work. Less typing, more reviewing. Less problem-solving, more problem-defining. You become a code reviewer who occasionally writes code, rather than a coder who occasionally reviews.

Is that better? Ask me in six months.

For now: update to Xcode 26.3, set up an agent, try it on a side project. The learning curve is about a day. The productivity gain is real. The "AI took my job" fears are premature.

The agents are good interns. They're not senior engineers. Not yet.
