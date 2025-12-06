# PR1.10 Remaining Parity Features

**Status**: Planning  
**Purpose**: Capture the few parity gaps not covered by PR1.8 or PR1.9.

---

## Overview

PR1.8 covers 5 core items; PR1.9 adds 28 more. Only a couple of gaps remain from the research doc that were not explicitly captured in PR1.9.

---

## P2: Conversion File Updates (Real-time)  
**Scope**: Add real-time conversion event handling to the AI SDK path.  
**What**: Handle Centrifugo events for conversion status and per-file updates so the UI reflects conversion progress live.  
**Main Path Reference**: `useCentrifugo.ts` handlers `handleConversionFileUpdatedMessage` and `handleConversationUpdatedMessage`.  
**UI Dependency**: Feeds `ConversionProgress` (already listed in PR1.9) and conversionsAtom hydration.  
**Complexity**: Medium  
**Est. Hours**: 3-4

---

## Notes
- All other research-identified gaps are already captured in PR1.9’s inventory (plan UI/actions, role selector, rollback, per-file accept/reject, terminal, command menu, feedback, etc.).
- This file stays small by design; it should only track what PR1.9 did not explicitly enumerate.

*Last Updated: Dec 5, 2025*
{
  "cells": [],
  "metadata": {
    "language_info": {
      "name": "python"
    }
  },
  "nbformat": 4,
  "nbformat_minor": 2
}