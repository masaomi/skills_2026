# skills_2026 Submodule Usage Memo (EN / JA)

> This memo explains how to use **skills_2026** as a Git submodule across projects.
> It includes the **recommended directory layout**, **clone/setup commands**, **update workflow**, and **common pitfalls**.

---

## English

## 1. What is `skills_2026`?

`skills_2026` is a shared knowledge base repository intended to act like reusable “SKILLs” across all projects.
Each project includes this repository as a **Git submodule**, so the project can consistently reference the same version of shared knowledge.

Repository:

* `git@github.com:masaomi/skills_2026.git`

---

## 2. Recommended location in each project

Pick one location and keep it consistent across projects.
Recommended:

* `docs/skills_2026/`

Example:

```
<project-root>/
  docs/
    skills_2026/   # submodule
```

---

## 3. Add `skills_2026` as a submodule

From the project root:

```bash
git submodule add git@github.com:masaomi/skills_2026.git docs/skills_2026
git commit -m "Add skills_2026 as submodule"
```

This will create/update:

* `.gitmodules`
* the submodule pointer (a specific commit reference)

---

## 4. Clone a project *with* submodules (important)

### Option A: Clone including submodules

```bash
git clone --recurse-submodules <project-repo-url>
```

### Option B: If you already cloned without submodules

```bash
git submodule update --init --recursive
```

> If you forget this step, the `docs/skills_2026/` directory may appear empty.

---

## 5. Updating `skills_2026` inside a project

### Typical workflow

```bash
cd docs/skills_2026
# update skills repo

git pull origin main

cd ../..
# record the new submodule commit in the parent project

git add docs/skills_2026
git commit -m "Update skills_2026 submodule"
```

### Why this matters

A Git submodule is pinned to a **specific commit**.
This ensures:

* reproducibility (everyone uses the same skills version)
* controlled upgrades (projects can adopt updates when ready)

---

## 6. Common pitfalls

1. **Submodule content is missing after clone**

* Fix with:

  ```bash
  git submodule update --init --recursive
  ```

2. **You updated `skills_2026`, but forgot to commit the pointer in the parent project**

* You must commit changes in the parent repo:

  ```bash
  git add docs/skills_2026
  git commit -m "Update skills_2026 submodule"
  ```

3. **Detached HEAD inside submodule**

* Submodules commonly sit at a commit, not a branch.
* If you want to work inside `skills_2026` itself, open that repo separately and commit there.

---

## 7. Suggested best practices

* Keep `skills_2026` files **small and modular** (1 skill = 1 file).
* Maintain an `INDEX.md` at the repo root to reduce noise and help Cursor/LLMs find relevant skills.
* In Cursor rules/instructions, prefer: “Use `INDEX.md` as the entry point; read only what’s needed.”

---

## 8. Quick commands cheat sheet

### Add submodule

```bash
git submodule add git@github.com:masaomi/skills_2026.git docs/skills_2026
```

### Clone with submodules

```bash
git clone --recurse-submodules <repo-url>
```

### Init/update submodules

```bash
git submodule update --init --recursive
```

### Update skills within a project

```bash
cd docs/skills_2026 && git pull origin main
cd ../.. && git add docs/skills_2026 && git commit -m "Update skills_2026 submodule"
```

---

---

## 日本語（Englishの後）

## 1. `skills_2026` とは？

`skills_2026` は、全プロジェクト共通で再利用できる知識を蓄積するためのリポジトリです。
Claude Code の “SKILLs” のように、設計指針・手順・チェックリスト・テンプレート等を集約し、各プロジェクトから **Git submodule** として参照します。

リポジトリ：

* `git@github.com:masaomi/skills_2026.git`

---

## 2. 各プロジェクトでの配置場所（推奨）

プロジェクト間で場所を統一しておくと運用が楽です。
推奨：

* `docs/skills_2026/`

例：

```
<project-root>/
  docs/
    skills_2026/   # submodule
```

---

## 3. `skills_2026` を submodule として追加する

プロジェクトルートで以下を実行します：

```bash
git submodule add git@github.com:masaomi/skills_2026.git docs/skills_2026
git commit -m "Add skills_2026 as submodule"
```

この操作により、親リポジトリには以下が記録されます：

* `.gitmodules`
* submodule が参照する **特定コミットID**（＝このプロジェクトで使う skills のバージョン）

---

## 4. submodule 付きでプロジェクトを clone する（重要）

### 方法A：submodule も一緒に clone

```bash
git clone --recurse-submodules <project-repo-url>
```

### 方法B：すでに clone 済みの場合（submodule が空の時）

```bash
git submodule update --init --recursive
```

> この手順を忘れると、`docs/skills_2026/` が空のように見えることがあります。

---

## 5. プロジェクト内で `skills_2026` を更新する手順

### 典型的な更新フロー

```bash
cd docs/skills_2026
# skills の更新

git pull origin main

cd ../..
# 親リポジトリ側に submodule の参照コミット更新を記録

git add docs/skills_2026
git commit -m "Update skills_2026 submodule"
```

### なぜこれが必要？

submodule は常に **特定コミット**に固定されます。
そのため：

* 再現性：誰が clone しても同じ skills 版になる
* 統制：各プロジェクトが「いつ更新を取り込むか」を管理できる

---

## 6. よくある落とし穴

1. **clone したのに submodule の中身が無い**

* 以下で初期化してください：

  ```bash
  git submodule update --init --recursive
  ```

2. **skills を更新したのに、親リポジトリで submodule の更新を commit していない**

* 親リポジトリ側で以下を忘れず実行：

  ```bash
  git add docs/skills_2026
  git commit -m "Update skills_2026 submodule"
  ```

3. **submodule が detached HEAD になっている**

* submodule はコミット固定で運用されがちです。
* `skills_2026` 自体を編集・開発する場合は、`skills_2026` を単独リポジトリとして扱い、そちらでコミットしてください。

---

## 7. 推奨ベストプラクティス

* `skills_2026` は **小さく分割**して管理（1 skill = 1 file）。
* ルートに `INDEX.md` を用意し、どこに何があるかを一覧化。
* Cursor のルールには「INDEX.md を起点に、必要なスキルだけ読む」方針を書くと、コンテキスト消費が抑えられます。

---

## 8. すぐ使えるコマンド集

### submodule 追加

```bash
git submodule add git@github.com:masaomi/skills_2026.git docs/skills_2026
```

### submodule 付き clone

```bash
git clone --recurse-submodules <repo-url>
```

### submodule 初期化・更新

```bash
git submodule update --init --recursive
```

### プロジェクト内で skills 更新

```bash
cd docs/skills_2026 && git pull origin main
cd ../.. && git add docs/skills_2026 && git commit -m "Update skills_2026 submodule"
```

