---
description: Create a comprehensive strategic plan with structured task breakdown
argument-hint: Describe what you need planned (e.g., "refactor authentication system", "implement microservices")
---

## 繁體中文說明 | Traditional Chinese Documentation

**命令名稱：** `/dev-docs`

**用途：** 創建具有結構化任務分解的綜合戰略計劃

**何時使用：**
- 需要規劃複雜的重構項目（例如：重構認證系統）
- 實施新的架構模式（例如：微服務架構）
- 開始大型功能開發之前
- 需要詳細的任務分解和時間線估算

**輸出內容：**
該命令會在 `dev/active/[任務名稱]/` 目錄下創建三個文件：
1. `[任務名稱]-plan.md` - 綜合計劃文檔
2. `[任務名稱]-context.md` - 關鍵文件、決策和依賴關係
3. `[任務名稱]-tasks.md` - 任務檢查清單格式，用於跟踪進度

**計劃內容包括：**
- 執行摘要
- 當前狀態分析
- 建議的未來狀態
- 實施階段（分為多個部分）
- 詳細任務（具有明確驗收標準的可操作項目）
- 風險評估和緩解策略
- 成功指標
- 所需資源和依賴關係
- 時間線估算

**使用示例：**
```
/dev-docs 重構認證系統以使用 OAuth 2.0
/dev-docs 實施微服務架構並遷移現有功能
/dev-docs 添加多租戶支持到現有應用
```

**最佳實踐：**
- 在退出計劃模式後使用此命令，當您對需要完成的內容有清晰的願景時
- 創建的任務結構在上下文重置後仍然存在
- 計劃必須是自包含的，包含所有必要的上下文

---

You are an elite strategic planning specialist. Create a comprehensive, actionable plan for: $ARGUMENTS

## Instructions

1. **Analyze the request** and determine the scope of planning needed
2. **Examine relevant files** in the codebase to understand current state
3. **Create a structured plan** with:
   - Executive Summary
   - Current State Analysis
   - Proposed Future State
   - Implementation Phases (broken into sections)
   - Detailed Tasks (actionable items with clear acceptance criteria)
   - Risk Assessment and Mitigation Strategies
   - Success Metrics
   - Required Resources and Dependencies
   - Timeline Estimates

4. **Task Breakdown Structure**: 
   - Each major section represents a phase or component
   - Number and prioritize tasks within sections
   - Include clear acceptance criteria for each task
   - Specify dependencies between tasks
   - Estimate effort levels (S/M/L/XL)

5. **Create task management structure**:
   - Create directory: `dev/active/[task-name]/` (relative to project root)
   - Generate three files:
     - `[task-name]-plan.md` - The comprehensive plan
     - `[task-name]-context.md` - Key files, decisions, dependencies
     - `[task-name]-tasks.md` - Checklist format for tracking progress
   - Include "Last Updated: YYYY-MM-DD" in each file

## Quality Standards
- Plans must be self-contained with all necessary context
- Use clear, actionable language
- Include specific technical details where relevant
- Consider both technical and business perspectives
- Account for potential risks and edge cases

## Context References
- Check `PROJECT_KNOWLEDGE.md` for architecture overview (if exists)
- Consult `BEST_PRACTICES.md` for coding standards (if exists)
- Reference `TROUBLESHOOTING.md` for common issues to avoid (if exists)
- Use `dev/README.md` for task management guidelines (if exists)

**Note**: This command is ideal to use AFTER exiting plan mode when you have a clear vision of what needs to be done. It will create the persistent task structure that survives context resets.