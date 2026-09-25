# Daily Dev Workflow — Quick Reference

**Access:** SSH into VM2, code in the browser via code-server.

```powershell
ssh -i C:\Users\<you>\.ssh\id_ed25519 devbox@10.224.20.4
```
Quick access
```powershell
ssh vm2-dev
```

code-server (browser editor): `https://10.224.20.4:8443`

> Replace `<name>` everywhere with your own short identifier (assigned by the team lead).

---

## Two working folders — pick the right one

| | `~/workspace/scratch/<name>/` | `~/workspace/worktrees/<name>/` |
|---|---|---|
| Use for | Quick experiments, or while your git isn't set up yet | Real work that goes into the project |
| Needs git? | No | Yes, required |
| Commit/push/PR? | No | Yes |
| Gets shared code from `main`? | No | Yes |

**Rule:** everything git-related (branches, commits, push, PRs) only happens in `worktree`. If you find yourself typing `git` inside `scratch`, it's time to move that work to a worktree.

---

## Scratch — no git needed

```bash
mkdir -p ~/workspace/scratch/<name>  --- tạo folder
docker exec dbt-dlt python3 /workspace/scratch/<name>/<file>.py  -----lệnh chạy file py, trong dự án này lúc chạy file load_oulad cần thêm --env-file như câu lệnh dưới
docker exec --env-file ~/.dbt_env_<tên> dbt-dlt python3 /workspace/worktrees/<tên-dev>/dlt_pipelines/load_oulad.py
```

Edit via code-server. Can't commit, can't push, can't pull shared code — just a personal sandbox.

---

## Worktree — real contributions

### First-time setup (once)

**1. Check your git works:**
```bash
cd ~/workspace/vn-data-governance
git pull
```
If it asks for username/password repeatedly or fails with `403`: create a GitHub Personal Access Token, then:
```bash
git config user.name "Your Name"
git config user.email "you@scc.com"
git config --global credential.helper store
```
Use the PAT as the password when prompted. Re-run `git pull` — should work cleanly now.

**2. Get your personal env file** (decides which schema you write to):
```bash
ls -la ~/.dbt_env_*
```
<img width="503" height="104" alt="image" src="https://github.com/user-attachments/assets/7c949b9d-e69d-4dcd-b5b3-3951a24b4377" />

If missing, ask the team lead, or self-create:
```bash
~/infra/scripts/dbt-env.sh create <name>
```

**3. Create your worktree:**
```bash
mkdir -p ~/workspace/worktrees   --- tạo thư mục-p = parents: Cho phép tạo luôn các thư mục cha nếu chúng chưa tồn tại, nếu thư mục đã tồn tại: Không báo lỗi, Không ghi đè (overwrite) thư mục, Không xóa file bên trong, Không thay đổi nội dung hiện có

cd ~/workspace/vn-data-governance
git checkout main && git pull    ----checkout: Lệnh chuyển branch hiện tại sang branch main/ git pull: kéo code mới nhất từ GitHub về.

git worktree add ../worktrees/<name> -b feature/your-task origin/main
sudo chown -R $(id -u):$(id -g) ~/workspace/worktrees/<name>
mkdir -p ~/workspace/worktrees/<name>/.dlt
cp ~/workspace/vn-data-governance/.dlt/secrets.toml ~/workspace/worktrees/<name>/.dlt/secrets.toml
```

**4. Verify:**
```bash
docker exec --env-file ~/.dbt_env_<name> dbt-dlt dbt debug --project-dir /workspace/worktrees/<name>/dbt_project
```
Expect `All checks passed!`. If `dbt_project.yml` is missing, someone on the team needs to create and merge it first — ask in chat.

### Daily flow

```bash
# 1. SSH in, cd into your worktree (or create a new one for new work)
cd ~/workspace/worktrees/<name>

# 2. Edit code via code-server

# 3. Run dlt
docker exec dbt-dlt python3 /workspace/worktrees/<name>/dlt_pipelines/pipeline_sample.py

# 4. Run dbt
docker exec --env-file ~/.dbt_env_<name> dbt-dlt dbt build --target dev --project-dir /workspace/worktrees/<name>/dbt_project

# 5. Check results
psql -h 10.224.20.3 -U dbt_dev_role -d dwh_dev -c "SELECT * FROM dbt_<name>_gold.dim_employee LIMIT 5;"

# 6. Commit, push, open a PR
git status   # confirm no .env / secrets.toml
git add . && git commit -m "feat: ..." && git push origin feature/your-task
```

Open a PR → review → merge. Dagster picks up merged code automatically on its next scheduled run — nothing else to do.

---

## Cheat sheet

| Task | Command |
|---|---|
| SSH in | `ssh -i C:\Users\<you>\.ssh\id_ed25519 devbox@10.224.20.4` |
| Check git auth | `cd ~/workspace/vn-data-governance && git pull` |
| New worktree | `cd ~/workspace/vn-data-governance && git worktree add ../worktrees/<name> -b feature/x origin/main` |
| List worktrees | `cd ~/workspace/vn-data-governance && git worktree list` |
| Remove worktree | `cd ~/workspace/vn-data-governance && git worktree remove ../worktrees/<name>` |
| Verify your schema | `docker exec --env-file ~/.dbt_env_<name> dbt-dlt env \| grep DBT_USER` |
| dbt debug/run/test/build | `docker exec --env-file ~/.dbt_env_<name> dbt-dlt dbt <cmd> --target dev --project-dir /workspace/worktrees/<name>/dbt_project` |
| Run a dlt script | `docker exec dbt-dlt python3 /workspace/worktrees/<name>/dlt_pipelines/<file>.py` |
| Query Postgres | `psql -h 10.224.20.3 -U dbt_dev_role -d dwh_dev` |

**Superset:** `https://10.224.20.5:8088` — **OpenMetadata:** `https://10.224.20.5:8585` — browse only, no SSH needed.

---

## FAQ

**Why `docker exec` instead of running `python3`/`dbt` directly?**
Those tools aren't installed on the VM2 host — only inside the `dbt-dlt` container.

**Why not just work in `~/workspace/vn-data-governance`?**
It's shared by the whole team and is what Dagster reads for its nightly build. Checking out a different branch there breaks things for everyone. Worktrees keep each person's work physically isolated.

**Forgot `--env-file`?**
Your command still runs, but writes to the `shared` schema instead of yours — easy to overwrite teammates' data. Always double-check with `env | grep DBT_USER`.

**New to the team — where do I start?**
SSH in → read the scratch vs. worktree table above → follow "First-time setup" → then use the daily flow.
