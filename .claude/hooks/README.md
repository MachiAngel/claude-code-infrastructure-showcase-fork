# Hooks | 鉤子

Claude Code hooks that enable skill auto-activation, file tracking, and validation.

---

## 繁體中文說明 | Traditional Chinese Documentation

本文檔介紹 Claude Code 的鉤子（Hooks）系統。鉤子是在 Claude 工作流的特定時間點運行的腳本。

**核心概念：**
鉤子可以在以下時機執行：
- **UserPromptSubmit**：當用戶提交提示時
- **PreToolUse**：在工具執行之前
- **PostToolUse**：在工具完成之後
- **Stop**：當用戶請求停止時

**關鍵洞察：**
鉤子可以修改提示、阻止操作和跟踪狀態 - 實現 Claude 單獨無法完成的功能。

**必要的鉤子（從這裡開始）：**

1. **skill-activation-prompt（UserPromptSubmit）**
   - 目的：根據用戶提示和文件上下文自動建議相關技能
   - 工作原理：
     * 讀取 skill-rules.json
     * 將用戶提示與觸發模式匹配
     * 檢查用戶正在處理的文件
     * 將技能建議注入 Claude 的上下文
   - 這是使技能自動激活的核心鉤子

2. **post-tool-use-tracker（PostToolUse）**
   - 目的：跟踪文件更改以在會話之間保持上下文
   - 工作原理：
     * 監控 Edit/Write/MultiEdit 工具調用
     * 記錄哪些文件被修改
     * 為上下文管理創建緩存
     * 自動檢測項目結構

**可選鉤子（需要定制）：**

3. **tsc-check（Stop）** - TypeScript 編譯檢查
   - ⚠️ 警告：為多服務 monorepo 結構配置
   - 需要根據您的項目結構進行大量定制

4. **trigger-build-resolver（Stop）** - 編譯失敗時自動啟動錯誤解決代理
   - 依賴於 tsc-check 鉤子正常工作

**集成說明：**
1. 始終從兩個必要的鉤子開始
2. 添加 Stop 鉤子之前要詢問用戶 - 配置錯誤可能會阻塞
3. 確保腳本具有執行權限（chmod +x）
4. 在 settings.json 中正確配置

---

## What Are Hooks?

Hooks are scripts that run at specific points in Claude's workflow:
- **UserPromptSubmit**: When user submits a prompt
- **PreToolUse**: Before a tool executes  
- **PostToolUse**: After a tool completes
- **Stop**: When user requests to stop

**Key insight:** Hooks can modify prompts, block actions, and track state - enabling features Claude can't do alone.

---

## Essential Hooks (Start Here)

### skill-activation-prompt (UserPromptSubmit)

**Purpose:** Automatically suggests relevant skills based on user prompts and file context

**How it works:**
1. Reads `skill-rules.json`
2. Matches user prompt against trigger patterns
3. Checks which files user is working with
4. Injects skill suggestions into Claude's context

**Why it's essential:** This is THE hook that makes skills auto-activate.

**Integration:**
```bash
# Copy both files
cp skill-activation-prompt.sh your-project/.claude/hooks/
cp skill-activation-prompt.ts your-project/.claude/hooks/

# Make executable
chmod +x your-project/.claude/hooks/skill-activation-prompt.sh

# Install dependencies
cd your-project/.claude/hooks
npm install
```

**Add to settings.json:**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

**Customization:** ✅ None needed - reads skill-rules.json automatically

---

### post-tool-use-tracker (PostToolUse)

**Purpose:** Tracks file changes to maintain context across sessions

**How it works:**
1. Monitors Edit/Write/MultiEdit tool calls
2. Records which files were modified
3. Creates cache for context management
4. Auto-detects project structure (frontend, backend, packages, etc.)

**Why it's essential:** Helps Claude understand what parts of your codebase are active.

**Integration:**
```bash
# Copy file
cp post-tool-use-tracker.sh your-project/.claude/hooks/

# Make executable
chmod +x your-project/.claude/hooks/post-tool-use-tracker.sh
```

**Add to settings.json:**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
          }
        ]
      }
    ]
  }
}
```

**Customization:** ✅ None needed - auto-detects structure

---

## Optional Hooks (Require Customization)

### tsc-check (Stop)

**Purpose:** TypeScript compilation check when user stops

**⚠️ WARNING:** Configured for multi-service monorepo structure

**Integration:**

**First, determine if this is right for you:**
- ✅ Use if: Multi-service TypeScript monorepo
- ❌ Skip if: Single-service project or different build setup

**If using:**
1. Copy tsc-check.sh
2. **EDIT the service detection (line ~28):**
   ```bash
   # Replace example services with YOUR services:
   case "$repo" in
       api|web|auth|payments|...)  # ← Your actual services
   ```
3. Test manually before adding to settings.json

**Customization:** ⚠️⚠️⚠️ Heavy

---

### trigger-build-resolver (Stop)

**Purpose:** Auto-launches build-error-resolver agent when compilation fails

**Depends on:** tsc-check hook working correctly

**Customization:** ✅ None (but tsc-check must work first)

---

## For Claude Code

**When setting up hooks for a user:**

1. **Read [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)** first
2. **Always start with the two essential hooks**
3. **Ask before adding Stop hooks** - they can block if misconfigured  
4. **Verify after setup:**
   ```bash
   ls -la .claude/hooks/*.sh | grep rwx
   ```

**Questions?** See [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)
