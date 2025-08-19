
高通量 **多铁材料计算流程**，涵盖从 **结构初筛 → VASP 批量计算 → 高精度精算 → 最终判定** 的全链条。

---

## 📌 全流程

```mermaid
flowchart TD
  A[准备 CIF & 配置 config.yaml] --> B[初筛: screen_structures.py]
  B --> C[多铁启发式: multiferroic_screen.py]
  C --> D[生成 VASP 输入: submit_vasp.py (stage1..6)]
  D --> E[生成 PBS: generate_run_all_pbs.py]
  E --> F[批量计算: qsub run_all_progress.pbs]
  F --> G[汇总结果: collect_results.py / merge_results.py]
  G --> H{挑选候选?}
  H -- 是 --> I[生成 stage7_refine 高精度集]
  I --> J[运行 stage7 & 最终判定]
  H -- 否 --> J
```

---

## 🔹 0) 准备与检查

配置 `config.yaml`：

```yaml
vasp:
  exe: /public/apps/vasp/vasp_std        # VASP 可执行文件
  potcar_dir: /public/home/xsy/PAW_GGA_PW91
pbs:
  account: YOUR_ACCOUNT                  # PBS 项目号
```

快速检查：

```bash
which /public/apps/vasp/vasp_std
python -m scripts.check_potcar_db --cif-dir /public/home/xsydata/cifs_relaxed --potcar-dir /public/home/xsy/PAW_GGA_PW91
```

---

## 🔹 1) 初筛

```bash
python -m scripts.screen_structures   --cif-dir /public/home/xsydata/cifs_relaxed   --out all_screen.csv   --limit 0
```

输出：`all_screen.csv`（formula, nsites, density…）

---

## 🔹 2) 多铁启发式筛选

```bash
python -m scripts.multiferroic_screen   --cif-dir data/cifs_relaxed   --out multiferroic_screen.csv   --merge-all-screen
```

输出：
- `multiferroic_screen.csv` （候选标记）
- `all_screen_with_multiferroic.csv` （合并表）

---

## 🔹 3) 生成 VASP 输入 (stage1..6)

```bash
python -m scripts.submit_vasp   --cif-dir data/cifs_relaxed   --potcar-dir /public/home/xsy/PAW_GGA_PW91   --config config.yaml   --limit 0   --workdir work/vasp_runs
```

生成目录：

```
work/vasp_runs/gen_0/stage1_relax/
work/vasp_runs/gen_0/stage2_static/
...
```

---

## 🔹 4) PBS 脚本 & 提交计算

```bash
cd work/vasp_runs
python -m scripts.generate_run_all_pbs --config ../config.yaml --workdir . --out run_all.pbs
qsub run_all_progress.pbs
tail -f logs/progress.log
```

运行顺序：
```
stage1_relax → stage2_static → stage3_dos → stage4_band → stage5_soc_(001/010/100) → stage6_berry
```

---

## 🔹 5) 汇总结果

```bash
python -m scripts.collect_results   --workdir work/vasp_runs   --out relax_summary.csv   --best relax_best.csv

python -m scripts.merge_results   --screen all_screen.csv   --summary relax_summary.csv   --out all_screen_with_relax.csv
```

输出：
- `relax_summary.csv`
- `relax_best.csv`
- `all_screen_with_relax.csv`

---

## 🔹 6) 高精度 Stage7

```bash
python -m scripts.generate_stage7_refine   --workdir work/vasp_runs   --indices 0,2,5   --potcar-dir /public/home/xsy/PAW_GGA_PW91
```

目录：

```
gen_0/stage7_refine/opt/
gen_0/stage7_refine/mag/
gen_0/stage7_refine/polar/
gen_0/stage7_refine/soc/
```

运行方式：
- 在 `run_all_progress.pbs` 添加 stage7_refine
- 或手动运行

---

## 🔹 7) 最终判定

- **绝缘性**：gap > 0
- **极化**：Ps ≠ 0（Berry 或 stage7）
- **磁性**：FM/AFM 能量差明确
- **稳定性**：能量/带隙/极化对 +U 不敏感

---



