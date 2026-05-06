---
name: house-inventory
description: Manage home inventory and storage tracking with intelligent image indexing.
version: 0.1.0
category: productivity
storage_path: brain/concepts/house-inventory.md
---

# House Inventory

## Overview
House Inventory is a disciplined system for cataloging physical assets, tracking container contents (boxes, bags, suitcases), and maintaining a verifiable index of reference imagery. Designed for precision, speed, and consistency.

## Prerequisites
- **GBrain Setup:** This skill requires an initialized `gbrain` environment. If you do not have one, set it up first: 
  `gbrain init` or see [GBrain Repository](https://github.com/kongaharsha/hermes_stack).
- **Inventory File:** A valid markdown inventory file at the configured `storage_path`.

## Workflow
1. **Cataloging:** Extract objects/item descriptions from image uploads or manual text input.
2. **Container Indexing:** Identify the source/target container (e.g., Box #1).
3. **Validation:** Review extracted lists with the user to ensure terminology compliance before write operations.
4. **Persisting:** Use structured `patch` operations to append or update entries while maintaining file integrity.

## Format Principles
*   **Container-First Hierarchy:** Every item must belong to a defined container.
    ```markdown
    ### Container: [Name]
    *   **Type:** [Box/Bag/Suitcase]
    *   **Location:** [Storage/Home]
    *   **Reference Photos:** [link1], [link2]
    *   **Items:** [Standardized bulleted list]
    ```

## Standardized Terminology
*   **Consistency:** Use precise naming conventions (e.g., "Diyas" over "Mats"). 
*   **Renaming:** If descriptive terms are identified as inaccurate post-cataloging, perform an immediate refactor of the inventory entry to ensure future search accuracy.

## Reliability Patterns
*   **Atomic Patching:** All modifications to the inventory file must be done via `patch` operations to prevent data loss.
*   **Read-Before-Write:** Must read current `house-inventory.md` state prior to execution to prevent conflicts with other agents.

## Anti-Patterns
*   **Hardcoded Pathing:** Never assume absolute local file paths outside of the `storage_path` configuration.
*   **Vision Guessing:** Never assume item identity from low-confidence image analysis. If ambiguous, trigger a manual verification request to the user.
*   **Cluttered Indexing:** Do not create duplicate container entries; always verify existence before cataloging new items.
