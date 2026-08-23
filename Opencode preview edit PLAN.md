# OpenCode Desktop Agent-Touched File Editor

## Goal

Add a Cursor-like source editor to OpenCode Desktop for files OpenCode reads, creates, writes, patches, or references during a session.

This is not a full IDE or workspace file explorer. The feature focuses on files relevant to the active agent conversation.

## Product Scope

### Included

- A toggleable, resizable source-editor panel in OpenCode Desktop.
- Tabs for files touched by agent activity.
- Automatic tab opening for supported file-related tool calls:
  - Read
  - Write
  - Edit
  - Patch/apply-edit
  - Explicit file references where the session event stream exposes them
- Syntax-highlighted text editing with line numbers.
- Dirty-state tracking.
- Save and revert actions.
- Detection of external file changes.
- Diff view against the version before the agent's latest change.
- Highlighting and navigation for agent-modified line ranges.
- Workspace path validation.
- Integration with OpenCode snapshots/undo where supported.

### Excluded From First Release

- Workspace explorer or file tree.
- Global search across the repository.
- Multi-root workspaces.
- Full language-server editing features.
- Git staging, commit, branch, and merge UI.
- Collaborative editing.
- Approval-gating agent edits.
- Arbitrary file navigation unrelated to agent activity.

## Source Areas

Work from a local checkout of `anomalyco/opencode`.

Inspect and map responsibility across:

- `packages/desktop`
  - Electron main process
  - Renderer layout
  - Window state
  - IPC boundary
- `packages/app`
  - Shared web/desktop application behavior
  - Session state and routing
- `packages/session-ui`
  - Conversation and tool-call rendering
  - Session event subscriptions
- `packages/core`
  - File tools
  - Filesystem write/edit behavior
  - Snapshot and undo integration
- `packages/protocol`
  - Tool-call event schemas
  - Session event transport
- Existing editor, diff, syntax-highlighting, or file-viewing dependencies.

## Architecture

### Editor State

Create a desktop renderer store for the active session:

```ts
type OpenFile = {
  path: string
  language?: string
  source: "tool" | "reference" | "manual"
  toolCallID?: string
  openedAt: number

  originalContent: string
  originalVersion: FileVersion

  content: string
  currentVersion?: FileVersion

  dirty: boolean
  readOnly: boolean
  unsupportedReason?: string

  agentChanges?: Array<{
    startLine: number
    endLine: number
    toolCallID: string
  }>
}

type FileVersion = {
  hash: string
  mtimeMs?: number
  size?: number
}
```

State should also track:

- `activePath`
- Ordered open tabs
- Panel visibility
- Panel width
- Diff mode
- Pending external-change conflict
- Session-specific recent files

Persist only non-sensitive UI preferences, such as panel width and visibility. Do not persist unsaved editor contents across restarts in the first release.

### Tool Event Mapping

Subscribe to session events that represent completed or active tool calls.

For each file-related tool event:

1. Validate that the path resolves inside the workspace.
2. Ignore binary, missing, oversized, or unsupported files.
3. Read the current file content through an existing OpenCode backend API.
4. Add or update a tab.
5. Make the file active when:
   - The tool changed the file.
   - The user clicked the associated tool activity.
   - The file is not already visible and there is no active dirty tab conflict.
6. Record changed ranges when available from tool metadata.
7. Capture the pre-change version when a tool modifies the file.

Tool-event mapping must be centralized so new tool types can be registered without modifying panel components.

### File Access

The renderer must not use direct filesystem APIs.

Add or reuse IPC/server operations for:

- Read a workspace file.
- Read file metadata/version.
- Save a workspace file.
- Optionally retrieve a snapshot version for diff rendering.

All operations must:

- Resolve relative paths against the active workspace.
- Reject paths outside the workspace.
- Reject symlink escape paths after canonicalization.
- Enforce the same permission model used by OpenCode's existing read/edit tools.
- Return structured conflict data when the file version has changed.

### Save and Conflict Flow

On save:

1. Compare the file version captured when the tab was loaded with the current disk version.
2. If unchanged, save via the existing write/edit service.
3. Update content, original content, and version metadata.
4. Record the save in the same snapshot/audit mechanism as agent edits when feasible.
5. If changed externally, show a conflict dialog with:
   - Reload from disk
   - Compare changes
   - Overwrite disk after confirmation
   - Cancel

Avoid silently overwriting external changes.

### Agent Change Tracking

For files changed by OpenCode:

- Capture or obtain the pre-edit content.
- Compute changed line ranges using the existing diff implementation, if present.
- Decorate affected lines in the editor gutter or overview ruler.
- Provide previous/next agent change controls.
- Provide a diff mode comparing:
  - Original content before the latest agent edit
  - Current editor content or saved disk content

If a user edits the file after an agent change, maintain the original agent-change reference until the file is saved, reloaded, or a new agent change supersedes it.

## UI Design

### Layout

Use a three-region desktop layout:

- Conversation area: primary and unchanged by default.
- Editor panel: right side on wide windows.
- Optional narrow-window mode: editor replaces or overlays the conversation area with a back control.

The panel should be:

- Hidden by default until a relevant file is opened.
- Toggleable from the top-level session toolbar.
- Resizable.
- Restorable to its last width.

### Header

Include:

- `Files` label
- Open-file tab strip
- Dirty indicator on tabs
- Close action per tab
- Toggle for source/diff mode
- Save action
- Revert action
- File path with workspace-relative display
- External-change/conflict indicator

### Editor

Use an editor that supports:

- Solid-compatible rendering
- Syntax highlighting
- Line numbers
- Read-only mode
- Controlled or robust state synchronization
- Decorations for changed lines
- Keyboard shortcuts

Preferred choice: CodeMirror 6.

Use Monaco only if the repository already includes it or it provides a concrete integration advantage. CodeMirror has a smaller bundle and is typically simpler for an embedded editor panel.

### Keyboard Shortcuts

Define non-conflicting shortcuts after auditing current desktop bindings:

- Toggle file panel
- Save current file
- Close active file tab
- Toggle diff/source mode
- Next/previous agent change

Use platform-aware shortcuts.

## Implementation Steps

1. Clone OpenCode and install its workspace dependencies.
2. Run the desktop app before changes to establish a working baseline.
3. Trace tool-call events from `packages/core` through protocol/session state into the desktop renderer.
4. Identify existing file read/write, snapshot, and diff utilities.
5. Identify current desktop layout and session toolbar components.
6. Add typed file-editor state and actions in the renderer.
7. Add backend/IPC endpoints only where existing APIs cannot safely provide file content and version-aware saves.
8. Add a centralized file-tool event mapper.
9. Implement panel visibility, resizing, tabs, and session integration.
10. Integrate CodeMirror with supported file-type detection and read-only fallbacks.
11. Implement save/revert behavior with optimistic version checks.
12. Implement external-change detection:
    - On focus
    - Before save
    - After relevant tool events
    - Optionally through the existing workspace watcher
13. Implement agent-change decorations and diff mode.
14. Connect save operations to OpenCode snapshots/undo behavior if supported by core APIs.
15. Add error states for inaccessible, missing, binary, unsupported, and oversized files.
16. Add telemetry only if OpenCode's existing privacy and analytics conventions require it.
17. Update user documentation and release notes.

## Test Plan

### Unit Tests

- Workspace-relative path validation.
- Symlink/path traversal rejection.
- Tool-event-to-file-tab mapping.
- Deduplication of repeated file tool events.
- Dirty-state transitions.
- Version/hash conflict detection.
- Revert behavior.
- Agent change-range computation.
- Unsupported and binary-file classification.

### Component Tests

- Panel opens from an agent file event.
- A changed file becomes the active tab.
- User typing marks the tab dirty.
- Save clears dirty state.
- Revert restores original content.
- Tabs close correctly.
- Diff/source mode toggles correctly.
- External-change conflict controls render and execute correctly.
- Agent-change navigation focuses the expected line range.

### Integration Tests

- Start Desktop against a fixture workspace.
- Trigger an OpenCode read tool call and verify a read-only file tab opens.
- Trigger an edit/write tool call and verify the modified file tab opens with change markers.
- Edit and save through the panel; verify the disk file changes.
- Trigger a subsequent agent read; verify it receives the user-saved content.
- Modify the file externally; verify save detects and surfaces a conflict.
- Verify paths outside the workspace cannot be opened or saved.

### Manual Verification

- macOS desktop packaging and runtime.
- Linux and Windows behavior if supported by the existing desktop CI matrix.
- Large-file fallback behavior.
- Files with unsupported encodings.
- Multiple rapid agent writes to the same file.
- Session switching with dirty tabs.
- Keyboard navigation and accessibility.
- High-DPI and narrow-window layouts.

## Acceptance Criteria

- When OpenCode reads or changes a supported workspace text file, the file can appear in the Desktop editor panel.
- The user can inspect and modify agent-touched files without leaving OpenCode Desktop.
- Save uses OpenCode's safe workspace-bound backend path, not renderer filesystem access.
- Unsaved changes are visible and never silently discarded.
- External file changes are detected before overwrite.
- Agent-modified regions are identifiable and reviewable in a diff view.
- The conversation remains the primary experience, and the editor panel can be hidden or resized.
- Unsupported, binary, oversized, or out-of-workspace files fail safely with a clear explanation.
- Existing session, tool, permission, snapshot, and undo workflows do not regress.

## Open Questions

- Does the current desktop renderer already expose the complete tool-event stream, including pre- and post-edit file content?
- Can manual editor saves be represented as normal OpenCode snapshot entries without impersonating an agent tool call?
- Should explicit `@file` references open files automatically, or only make them available through a click target?
- Should the editor panel be global across sessions or scoped entirely to the active session?
- What size threshold should switch a file to read-only or refuse loading?
- Should Markdown receive a rendered preview in the initial release, or remain source-only?
