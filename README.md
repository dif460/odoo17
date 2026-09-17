# Odoo 17 环境(富维金属件 MES 支撑框架)

本目录 `E:\Work\python\odoo17` 是 **Odoo 17.0 源码框架**,用于支撑生产业务代码目录
`E:\Work\python\fw_metal_parts` 的运行。

> **重要约定**:`E:\Work\python\fw_metal_parts` 下的业务代码是生产落地项目,**不允许修改**。
> 所有运行时调整(custom addons、配置、补丁)都应在本 odoo17 目录内完成。

---

## 一、项目结构

```
E:\Work\python\odoo17\                      # 本框架(Odoo 17.0 源码)
├── odoo\                                   # Odoo 核心源码
├── odoo-bin                                # 启动入口
├── odoo.conf                               # 主服务配置(8069 / fw_metal_parts 库 / 5434)
├── odoo1.conf                              # 第二套实例配置(8068 / fw_metal_parts_cc 库 / 5435)
├── requirements.txt                        # Python 依赖
├── .venv\                                  # 虚拟环境
├── data\                                   # 主服务 data_dir(运行时,含 filestore/sessions)
├── data_cc\                                # 第二套实例 data_dir
├── custom_addons\
│   └── fw_login_compat\                    # 登录页 providers 桥接模块
└── README.md                               # 本文档

E:\Work\python\fw_metal_parts\             # 生产业务代码(只读)
├── odoo_source_model\                     # 富维自定义模块
└── parts_source_model\                    # 富维自定义模块
```

> **注意**:`odoo-bin` 仅为启动入口(`import odoo.cli`),真正源码在 `odoo/` 目录。用 `python -m odoo -c xxx.conf` 等价。

---

## 二、环境要求

| 组件      | 版本/要求                                              |
|-----------|--------------------------------------------------------|
| Python    | 3.10 – 3.14(本机 3.12.3)                                |
| PostgreSQL| Docker 镜像 `postgres:17-alpine`                        |
| Odoo      | 17.0                                                   |

两套实例各自使用**独立的 PostgreSQL 数据库端口**(5434 / 5435)、**独立的 data_dir**(data / data_cc)、**独立的 HTTP 端口**(8069 / 8068),互不干扰。

---

## 三、启动步骤

### 1. 启动 PostgreSQL( Docker )

两套实例各自一个容器、互不干扰:

#### 主服务容器(容器名 `odoo17_fw`,端口 **5434**,数据库 `fw_metal_parts`)

```powershell
docker run -d `
  --name odoo17_fw `
  -p 5434:5432 `
  -e POSTGRES_USER=odoo `
  -e POSTGRES_PASSWORD=odoo `
  -e POSTGRES_DB=fw_metal_parts `
  -v odoo17_fw_pgdata:/var/lib/postgresql/data `
  --restart unless-stopped `
  postgres:17-alpine
```

#### 第二套实例容器(容器名 `odoo17_fw_cc`,端口 **5435**,数据库 `fw_metal_parts_cc`)

```powershell
docker run -d `
  --name odoo17_fw_cc `
  -p 5435:5432 `
  -e POSTGRES_USER=odoo `
  -e POSTGRES_PASSWORD=odoo `
  -e POSTGRES_DB=fw_metal_parts_cc `
  -v odoo17_fw_cc_pgdata:/var/lib/postgresql/data `
  --restart unless-stopped `
  postgres:17-alpine
```

容器已存在时直接 `docker start odoo17_fw` / `docker start odoo17_fw_cc`。

验证:

```powershell
# 5434
docker exec odoo17_fw pg_isready -U odoo -h localhost -p 5432
# 5435
docker exec odoo17_fw_cc pg_isready -U odoo -h localhost -p 5432
```

> **自建容器命令模板**:若要新建第三套实例,只需替换容器名、端口、POSTGRES_DB、卷名即可。

### 2. 创建虚拟环境并安装依赖

```powershell
cd E:\Work\python\odoo17
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

除 `requirements.txt` 外,以下为业务模块实际需要、但未列在 requirements 中的第三方包(一次性安装):

```powershell
pip install openpyxl captcha chinese_calendar pycryptodome dingtalk-sdk
pip install alibabacloud_dyvmsapi20170525 alibabacloud_tea_openapi alibabacloud_tea_util
```

> `pymssql` / `pyodbc`(HRIS 同步用)在业务代码中是函数级惰性导入,启动不强制依赖;按需再装。

### 3. 初始化数据库并安装全部模块

包含 base + 全部 fw 业务模块 + 桥接模块,首次建库用 `-i`:

```powershell
cd E:\Work\python\odoo17
# 收集 fw 两目录下所有模块名,一次性安装
$mods = @(
  Get-ChildItem "E:\Work\python\fw_metal_parts\odoo_source_model" -Directory | Where-Object { Test-Path (Join-Path $_.FullName '__manifest__.py') } | Select-Object -ExpandProperty Name
  Get-ChildItem "E:\Work\python\fw_metal_parts\parts_source_model" -Directory | Where-Object { Test-Path (Join-Path $_.FullName '__manifest__.py') } | Select-Object -ExpandProperty Name
)
$modList = ($mods -join ',')

# 主服务:fw_metal_parts 库(5434 端口容器)
.\odoo-bin -c odoo.conf -d fw_metal_parts -i "base,$modList,fw_login_compat" --without-demo=all --stop-after-init

# 第二套实例:fw_metal_parts_cc 库(5435 端口容器)
# 注意 odoo1.conf 的 addons_path 默认不含 parts_source_model,装全模块前需先追加
.\odoo-bin -c odoo1.conf -d fw_metal_parts_cc -i "base,$modList" --without-demo=all --stop-after-init
```

若数据库已存在、仅需补齐/升级模块,用 `-u`:

```powershell
.\odoo-bin -c odoo.conf -d fw_metal_parts -u "$modList" --stop-after-init
```

> **`fw_metal_menu` 初始化需先建 3 个占位菜单记录**(因为该模块 `fw_menu_production_exec.xml` 引用 3 个不存在的 `ir.ui.menu` external id,导致 `ir_ui_menu.name` 为 NULL 报错)。可在安装它之前手动插入占位记录(见下方附录 A)。

### 4. 启动 Odoo 服务器

两套实例各自独立启动(不同配置文件、不同端口、不同 data_dir):

```powershell
cd E:\Work\python\odoo17

# 主服务(8069 端口, fw_metal_parts 库, 5434)
.\odoo-bin -c odoo.conf --dev=reload

# 第二套实例(8068 端口, fw_metal_parts_cc 库, 5435)
.\odoo-bin -c odoo1.conf --dev=reload
```

也可以用等价的 `python -m odoo -c xxx.conf` 形式。正常启动后日志会出现 `HTTP service (werkzeug) running on ...`。

---

## 四、data_dir 说明

`odoo.conf` 中的 `data_dir` 是 **Odoo 非数据库数据的根目录**(PostgreSQL 存不下或不适合存的东西都在这里):

```
data\                                     # 主服务 data_dir
├── addons\17.0\                          # 通过后台"应用商店"安装的模块
├── filestore\                            # 各数据库的附件(实际文件内容)
│   ├── fw_metal_parts\                   # fw_metal_parts 库的附件
│   │   ├── 01\..ff\                     # 按文件名哈希分桶存储
│   │   └── ...
│   ├── fw_metal_parts_cc\                # fw_metal_parts_cc 库的附件
│   └── ...
└── sessions\                            # 用户登录会话(退出即失效)
```

### 关键点

| 存储位置 | 存什么 | 备份/恢复 |
|----------|--------|-----------|
| PostgreSQL(`pg_dump`/`pg_restore`) | 业务表结构 + `ir_attachment` 元数据(附件 id/path) | 走 pg_dump |
| `data_dir\filestore\` | **附件实际内容**(PDF/Excel/图片/JS·CSS 打包文件) | **手动复制目录**,pg_dump 不会带上 |

> ⚠️ **恢复 Odoo 数据库不能只恢复一半**:pg_dump 还原表结构后,必须同时拷贝原来的 `filestore\<库名>` 目录,否则 Odoo 找不到附件文件,前端静态资源(JS/CSS)缺失会导致**白屏**。

---

## 五、访问地址

| 实例 | Web 后台 | 数据库直连 |
|------|----------|-----------|
| 主服务 | <http://localhost:8069> | host localhost,port 5434,db fw_metal_parts,user odoo,pwd odoo |
| 第二套实例 | <http://localhost:8068> | host localhost,port 5435,db fw_metal_parts_cc,user odoo,pwd odoo |

管理员账号默认 `admin`,密码由首次创建/设置。登录页为富维 MES 界面(含品牌、账号/密码、验证码)。

---

## 六、登录页修复说明(custom_addons / fw_login_compat)

登录页 500 的根因:

- fw 模块 `web_login_captcha_verification_code` 用 `@http.route('/web/login')` **覆写了登录控制器**,渲染上下文里未提供 `providers`。
- 而已安装模块 `auth_oauth`(被 fw 的 `dingtalk_login` 依赖,不能卸载)的 `login` 继承模板引用 `len(providers)`,`providers` 未定义时 QWeb 视为 `None` → `NoneType has no len()` → 500。

odoo17 侧的解法(不改任何 fw 文件):

- 新增桥接模块 `custom_addons/fw_login_compat`:
  - `depends: ['web_login_captcha_verification_code', 'auth_oauth']`
  - controller 继承 fw 的登录控制器,覆写 `/web/login`,在超类渲染前把 `providers` 兜底为 `[]`
- 将该模块目录追加到 `odoo.conf` 的 `addons_path`,并 `-i fw_login_compat` 安装。

效果:登录页仍采用 fw 设计(富维界面 + 验证码),且不再报错。

---

## 七、数据库备份与恢复

### 方式一:Odoo 后台备份(完整,推荐)

访问 `http://localhost:8069/web/database/manager`,点 **备份数据库**。这种方式会同时导出:
- PostgreSQL 表结构和数据
- **filestore 附件**(会打包进 dump 文件)

恢复也是这个页面,选 **恢复数据库**,上传 dump 即可。**恢复后自动包含 filestore**,不会出现白屏。

### 方式二:pg_dump/pg_restore(部分,需额外手动)

仅导出 PostgreSQL 表,**不包含 filestore**。恢复后需要手动拷贝 filestore:

```powershell
# 备份主服务数据库
docker exec odoo17_fw pg_dump -U odoo -d fw_metal_parts --format=custom -f /tmp/fw_metal_parts.dump
docker cp odoo17_fw:/tmp/fw_metal_parts.dump E:\Work\python\odoo17\fw_metal_parts.dump

# 恢复到新库(以恢复为 df 为例)
docker exec -it odoo17_fw psql -U odoo -d postgres -c "DROP DATABASE IF EXISTS df;"
docker exec -it odoo17_fw psql -U odoo -d postgres -c "CREATE DATABASE df OWNER odoo ENCODING 'UTF8' TEMPLATE template0;"
docker cp E:\Work\python\odoo17\fw_metal_parts.dump odoo17_fw:/tmp/df.dump
docker exec odoo17_fw pg_restore -U odoo -d df --no-owner --no-privileges /tmp/df.dump

# ⚠️ 关键:手动拷贝 filestore(否则前端静态资源缺失 → 白屏)
xcopy /e /i "E:\Work\python\odoo17\data\filestore\fw_metal_parts" "E:\Work\python\odoo17\data\filestore\df"
```

### 恢复失败的典型症状

| 症状 | 原因 | 解法 |
|------|------|------|
| **启动崩 `there is no unique constraint matching given keys for referenced table res_users`** | pg_restore 跳过了索引/约束恢复(部分 dump 文件格式兼容问题),库表数据有了但结构残缺 | **重走方式一**(Odoo 后台导出的 dump 不会有这问题),或换 `pg_restore -c --if-exists` |
| **浏览器白屏 / `GET /web/assets xxx 500`** | filestore 没拷贝,附件元数据存在但文件路径不存在 | 手动 `xcopy` 原来的 filestore 目录 |
| **正常启动但 assets 总是报错** | filestore 拷贝了但 Odoo 缓存了旧路径 | 删掉 `data\filestore\<新库名>\ir_attachment` 里 assets 相关记录,用 `--dev=all` 重启让 Odoo 自动重建 |

### 恢复后验证数据库完整性

```powershell
# 检查是否有表缺失主键约束
docker exec -it odoo17_fw psql -U odoo -d df -c "SELECT COUNT(*) FROM information_schema.tables t WHERE t.table_schema='public' AND NOT EXISTS (SELECT 1 FROM information_schema.table_constraints c WHERE c.table_schema='public' AND c.table_name=t.table_name AND c.constraint_type='PRIMARY KEY');"
```

结果应为 **0**。大于 0 说明恢复残缺,需重新备份/恢复。

---

## 八、常用维护命令

```powershell
# 停止 / 启动数据库容器
docker stop odoo17_fw; docker stop odoo17_fw_cc
docker start odoo17_fw; docker start odoo17_fw_cc

# 升级全部已安装模块
.\odoo-bin -c odoo.conf -d fw_metal_parts -u all --stop-after-init

# 升级指定模块
.\odoo-bin -c odoo.conf -d fw_metal_parts -u sparepart,odoo_dynamic_workflow --stop-after-init

# 备份数据库(PostgreSQL 层,不含 filestore)
docker exec odoo17_fw pg_dump -U odoo -d fw_metal_parts --format=custom -f /tmp/fw_metal_parts.dump
docker cp odoo17_fw:/tmp/fw_metal_parts.dump E:\Work\python\odoo17\fw_metal_parts.dump

# 检查数据库完整性(缺失主键的表数量)
docker exec -it odoo17_fw psql -U odoo -d fw_metal_parts -c "SELECT COUNT(*) FROM information_schema.tables t WHERE t.table_schema='public' AND NOT EXISTS (SELECT 1 FROM information_schema.table_constraints c WHERE c.table_schema='public' AND c.table_name=t.table_name AND c.constraint_type='PRIMARY KEY');"
```

---

## 九、附录 A:`fw_metal_menu` 初始化占位记录

仅在该库尚未安装 `fw_metal_menu` 时需要预先插入(使 `menu_mes_screen_staff_bind` / `menu_mes_screen_duty` / `menu_mes_screen_inspection` 三个 external id 指向已有菜单,避免 `ir_ui_menu.name` NULL 报错):

```powershell
# 将下列 SQL 存入文件并在容器内执行(docker cp + psql -f)
BEGIN;
-- 1) 占位 menu(父级 = MES 终端菜单 id 386,name 为 jsonb 翻译结构)
INSERT INTO ir_ui_menu (create_uid, create_date, write_uid, write_date, parent_id, name, sequence, active)
SELECT 1, now(), 1, now(), 386, '{"en_US": "menu_mes_screen_staff_bind"}'::jsonb, 10, false
WHERE NOT EXISTS (SELECT 1 FROM ir_model_data WHERE module='fw_metal_menu' AND name='menu_mes_screen_staff_bind');
INSERT INTO ir_ui_menu (create_uid, create_date, write_uid, write_date, parent_id, name, sequence, active)
SELECT 1, now(), 1, now(), 386, '{"en_US": "menu_mes_screen_duty"}'::jsonb, 10, false
WHERE NOT EXISTS (SELECT 1 FROM ir_model_data WHERE module='fw_metal_menu' AND name='menu_mes_screen_duty');
INSERT INTO ir_ui_menu (create_uid, create_date, write_uid, write_date, parent_id, name, sequence, active)
SELECT 1, now(), 1, now(), 386, '{"en_US": "menu_mes_screen_inspection"}'::jsonb, 10, false
WHERE NOT EXISTS (SELECT 1 FROM ir_model_data WHERE module='fw_metal_menu' AND name='menu_mes_screen_inspection');
-- 2) 注册 external id
INSERT INTO ir_model_data (create_uid, create_date, write_uid, write_date, module, name, model, res_id)
SELECT 1, now(), 1, now(), 'fw_metal_menu', 'menu_mes_screen_staff_bind', 'ir.ui.menu', m.id
FROM ir_ui_menu m WHERE m.name::text LIKE '%menu_mes_screen_staff_bind%'
  AND NOT EXISTS (SELECT 1 FROM ir_model_data WHERE module='fw_metal_menu' AND name='menu_mes_screen_staff_bind');
INSERT INTO ir_model_data (create_uid, create_date, write_uid, write_date, module, name, model, res_id)
SELECT 1, now(), 1, now(), 'fw_metal_menu', 'menu_mes_screen_duty', 'ir.ui.menu', m.id
FROM ir_ui_menu m WHERE m.name::text LIKE '%menu_mes_screen_duty%'
  AND NOT EXISTS (SELECT 1 FROM ir_model_data WHERE module='fw_metal_menu' AND name='menu_mes_screen_duty');
INSERT INTO ir_model_data (create_uid, create_date, write_uid, write_date, module, name, model, res_id)
SELECT 1, now(), 1, now(), 'fw_metal_menu', 'menu_mes_screen_inspection', 'ir.ui.menu', m.id
FROM ir_ui_menu m WHERE m.name::text LIKE '%menu_mes_screen_inspection%'
  AND NOT EXISTS (SELECT 1 FROM ir_model_data WHERE module='fw_metal_menu' AND name='menu_mes_screen_inspection');
COMMIT;
```

> 父菜单 id `386` 为 `menu_fw_metal_mes_app`(fw_metal_mes 的 MES 终端菜单)的 res_id;若该库中 id 不同,请先用
> `SELECT name, res_id FROM ir_model_data WHERE name='menu_fw_metal_mes_app' AND module='fw_metal_mes';` 查询替换。

---

## 十、FAQ / 排错

- **`python -m odoo` vs `odoo-bin`**:二者等价,都是 `import odoo.cli; odoo.cli.main()` 的薄入口。
- **登录页 500 / `NoneType has no len()`**:确认 `fw_login_compat` 已安装且其目录在 `addons_path` 中。
- **模块安装缺失依赖报错**:按第三节第 2 步补齐第三方包后重跑 `-i/-u`。
- **`ir_ui_menu` 的 name 为 jsonb(翻译结构)**:直连数据库插入菜单时 name 需用 `'{"en_US": "..."}'::jsonb`。
- **`data_dir` 是什么**:Odoo 非数据库数据(附件/JS·CSS 打包/会话)的根目录,详见第四节。两套实例的 `data_dir` 必须**独立**,否则 filestore 路径和缓存会互相污染。
- **pg_restore 后启动崩 `there is no unique constraint matching given keys for referenced table res_users`**:说明 pg_restore 跳过了约束恢复,库表残缺。**必须重走 Odoo 后台备份/恢复**,不能用浏览器"数据库管理"页面去恢复 pg_dump 出来的 dump(格式不兼容)。
- **恢复后浏览器白屏 / `GET /web/assets xxx 500`**:pg_dump **不带 filestore**(附件实际内容)。恢复后需手动 `xcopy` 原来的 `filestore\<原库名>` 目录到 `data_dir\filestore\<新库名>`,或删掉新库 `ir_attachment` 里 assets 相关记录让 Odoo 自动重建。
- **长表名警告 `odoo_workflow_fw_production_parameter_abnormal_approve_users_rel is too long`**:PostgreSQL 标识符上限 63 字符。根因在 `odoo_dynamic_workflow` 模块动态生成 Many2many 表名时没有截断。修复方式是让拼接后的表名超限时自动加 MD5 digest 缩短(改动 `models/odoo_workflow.py` 里 `relation_table` 拼接逻辑)。
- **`sparepart` 模块 computed field 警告 `inconsistent 'store' for computed fields`**:同一 `compute` 方法下有字段 `store=True` 和 `store=False` 混用。`odoo17\parts_source_model\sparepart\models\stock_quant.py` 里 `max_qty`/`min_qty`(无 store)和 `ABC`(store=True)都用 `_compute_max_min_qty`,会导致读非存储字段时顺带重写存储字段。建议要么给 `max_qty`/`min_qty` 也加 `store=True`,要么拆成两个 compute 方法。
