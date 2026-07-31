# UMA4DLP

Unified Materials Archive for Deep Learning Potential — 用於 DLP 訓練的 unified materials 數據收集工具。

本工具集用於從 Multiple sources（FairChem、GPT-fake DFT 等）收集訓練數據，並提交至 Slurm 叢集進行 DFT 標籤計算。

---

## 程式功能說明

| 腳本 | 功能 |
|---|---|
| `arrange_data4UMA.pl` | 整理 UMA 數據為可用格式 |
| `check_UMAjobs.pl` | 檢查所有 UMA DFT 工作狀態 |
| `fairchem_modified.sh` | FairChem 數據修改腳本 |
| `gptfakeQE.py` | 使用 GPT 生成假 QE 輸出 |
| `submit_allslurm_sh.pl` | 批量提交 Slurm 工作 |
| `submit_sh4allDead.pl` | 重新提交 Dead 狀態的工作 |

---

## 依賴環境

| 項目 | 需求 |
|---|---|
| 語言 | Perl 5.x, Python 3.x |
| DFT | Quantum ESPRESSO 或 FairChem |
| 排程 | Slurm |

---

## 使用方法

```bash
# 整理數據
perl arrange_data4UMA.pl

# 檢查工作狀態
perl check_UMAjobs.pl

# 批量提交
perl submit_allslurm_sh.pl

# 重新提交失敗的工作
perl submit_sh4allDead.pl
```

---

## AI Agent 操控指南

```
任務: 從 UMA 收集並標籤數據
步驟:
1. perl arrange_data4UMA.pl 整理數據
2. perl submit_allslurm_sh.pl 批量提交
3. perl check_UMAjobs.pl 監控進度
4. perl submit_sh4allDead.pl 重新提交失敗
5. 重複 3-4 直到所有 Done
```
