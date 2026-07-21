---
name: android-accessibility
description: Expert checklist and prompts for auditing and fixing Android accessibility issues, especially in Jetpack Compose. Use when reviewing a screen or component for accessibility, adding content descriptions or semantics, fixing touch targets or contrast, or preparing an app for TalkBack users.
---

# Android Accessibility Checklist

## Instructions

Analyze the provided component or screen for the following accessibility aspects.

### 1. Content Descriptions
*   **Check**: Do `Image` and `Icon` composables have a meaningful `contentDescription`?
*   **Decorative**: If an image is purely decorative, use `contentDescription = null`.
*   **Actionable**: If an element is clickable, the description should describe the *action* (e.g., "Play music"), not the icon (e.g., "Triangle").

### 2. Touch Target Size
*   **Standard**: Minimum **48x48dp** for all interactive elements.
*   **Fix**: Use `Modifier.minimumInteractiveComponentSize()` (Material 3) if the visual icon is smaller, or wrap in a `Box` with appropriate padding. The enforced size can be read/overridden via the `LocalMinimumInteractiveComponentSize` composition local — do not lower it below 48dp.

### 3. Color Contrast
*   **Standard**: WCAG AA requires **4.5:1** for normal text and **3.0:1** for large text/icons.
*   **Tool**: Verify colors against backgrounds using contrast logic.

### 4. Focus & Semantics
*   **Focus Order**: Ensure keyboard/screen-reader focus moves logically (e.g., Top-Start to Bottom-End).
*   **Grouping**: Use `Modifier.semantics(mergeDescendants = true)` for complex items (like a row with text and icon) so they are announced as a single item.
*   **State descriptions**: Use `stateDescription` to describe custom states (e.g., "Selected", "Checked") if standard semantics aren't enough.

### 5. Headings
*   **Traversal**: Mark title texts with `Modifier.semantics { heading() }` to allow screen reader users to jump between sections.

### 6. Custom Controls: `clearAndSetSemantics`
*   **When**: A custom control is built from several composables whose individual semantics would confuse TalkBack (e.g., a rating widget made of five icons, or a card that acts as one button).
*   **Fix**: Use `Modifier.clearAndSetSemantics { ... }` on the container to wipe descendant semantics and declare the merged meaning yourself (e.g., `contentDescription`, `role`, `stateDescription`, `onClick` label).
    ```kotlin
    Row(
        modifier = Modifier.clearAndSetSemantics {
            contentDescription = "Rating: $rating of 5 stars"
        }
    ) { /* five Icon composables */ }
    ```
*   **Caution**: This hides *all* descendant semantics — verify nothing interactive inside becomes unreachable. Prefer `semantics(mergeDescendants = true)` when the default merged announcement is adequate.

## TalkBack Testing Basics

Manual verification with TalkBack is the ground truth for screen-reader support:

1.  **Enable**: Settings > Accessibility > TalkBack (or `adb shell settings put secure enabled_accessibility_services com.google.android.marvin.talkback/com.google.android.marvin.talkback.TalkBackService`).
2.  **Navigate**: Swipe right/left to move focus linearly; double-tap to activate the focused element; use reading controls (swipe up/down) to jump by headings.
3.  **Verify**: Every interactive element is reachable, announcements are meaningful (action, not icon shape), grouped items are read as one, and state changes ("Selected", "Checked") are announced.
4.  **Automate a first pass** with the Accessibility Scanner app or `AccessibilityChecks.enable()` in Espresso tests, but do not treat them as a substitute for a TalkBack pass.

## Example Prompts for Agent Usage
*   "Analyze the content description of this Image. Is it appropriate?"
*   "Check if the touch target size of this button is at least 48dp."
*   "Does this custom toggle button report its 'Checked' state to TalkBack?"
