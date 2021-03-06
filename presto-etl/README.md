# presto-etl

平台目前架构如下:

- __OLAP引擎__ 为 [Presto](https://prestodb.io/)
- __调度工具__ 为 [Azkaban](https://azkaban.github.io/)
- 所有 __ETL__ 相关 __SQL__ 单独维护在远程 __Git 仓库__ 上
- 数仓的结构做了分层: `ods`, `cdm`, `ads`，SQL 脚本文件的命名为 `create`, `fully`, `loop`，对应 __建表__，__全量__，__增量__，
    这种结构同时体现在 __维护 SQL 脚本的 Git 仓库的目录结构上__

因此需要一个带参数的脚本型的工具来通过 __请求 Git 对应 SQL 的 url__，然后通过 __presto-python-client__ 发送到 __presto__ 执行，
azkaban 调度时只需配置好相关的参数即可

因为公司一些项目的 ETL 需求比较刁钻，而 presto 只能执行 SQL，因此脚本的参数会有些繁杂，特别是 `--placeholder.config` 参数，
有觉得不解的查看代码的注释即可

## help & usage

- first of all

```shell
# cd to presto-tools directory
> $ source venv/bin/activate
(venv) > $ cd presto-etl
```

- help:

```shell
(venv) > $ python3 presto-etl.py -h

usage: python3 presto-etl.py [-h] [--usage] [--presto.host PRESTO_HOST]
                             [--presto.port PRESTO_PORT]
                             [--presto.user PRESTO_USER]
                             [--presto.catalog PRESTO_CATALOG]
                             [--presto.schema PRESTO_SCHEMA]
                             [--sql.url.prefix SQL_URL_PREFIX]
                             [--sql.dir SQL_DIR]
                             [--sql.names [SQL_NAMES [SQL_NAMES ...]]]
                             [--placeholder.config [PLACEHOLDER_CONFIG [PLACEHOLDER_CONFIG ...]]]

This is a python etl script

optional arguments:
  -h, --help            show this help message and exit
  --usage               show usage
  --presto.host PRESTO_HOST
                        set presto host
  --presto.port PRESTO_PORT
                        set presto port
  --presto.user PRESTO_USER
                        set presto user
  --presto.catalog PRESTO_CATALOG
                        set presto catalog
  --presto.schema PRESTO_SCHEMA
                        set presto schema
  --sql.url.prefix SQL_URL_PREFIX
                        set the git repo url. (route to the system name [e.g.
                        crm, mms, tpos, etc.]) for sql file
  --sql.dir SQL_DIR     set the parent diretory for sql file (the dir name is
                        always be the table name.)
  --sql.names [SQL_NAMES [SQL_NAMES ...]]
                        set the sql file name for sql file, avaliable to
                        recieve multiple argment. (the sql name is always be
                        the function name, like create, fully, loop, etc.)
  --placeholder.config [PLACEHOLDER_CONFIG [PLACEHOLDER_CONFIG ...]]
                        set the placeholder config. (the format of this option
                        is <sql.name>:<placeholder.sql.name>. see the annotate
                        of function get_placeholder_config() for more detail
                        to use it)
```

- usage:

```shell
(venv) > $ python3 presto-etl.py --usage

python3 presto-etl.py <option> [arguments]

for help
--------
python3 presto-etl.py -h

example
-------
python3 presto-etl.py \
    --presto.host 10.10.22.5 \
    --presto.port 10300 \
    --presto.user dev \
    --presto.catalog dev_hive \
    --presto.schema ods_test \
    --sql.url.prefix http://gitlab.company.com/group/repo/raw/branch/sql/etl/dwh/ods/some_system \
    --sql.dir table_name \
    --sql.names create fully

example for azkaban properties
------------------------------
presto.host=10.10.22.5
presto.port=10300
presto.user=dev
presto.catalog=dev_hive
presto.schema=ods_test
git.branch=dev
sql.url.prefix=http://gitlab.company.com/group/repo/raw/{git.branch}/sql/etl/dwh/ods/some_system
sql.dir=table_name
sql.names=create fully

python.interpreter=/program/presto-tools/venv/bin/python3
python.etl.dir=/program/presto-tools/presto-etl

cmd=${python.interpreter} ${python.etl.dir}/presto-etl.py \
    --presto.host ${presto.host} \
    --presto.port ${presto.port} \
    --presto.user ${presto.user} \
    --presto.catalog ${presto.catalog} \
    --presto.schema ${presto.schema} \
    --sql.url.prefix ${sql.url.prefix} \
    --sql.dir ${sql.dir} \
    --sql.names ${sql.names}
```