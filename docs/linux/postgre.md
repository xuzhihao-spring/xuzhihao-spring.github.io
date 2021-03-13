# postgre命令

## 1.备份恢复

```bash
pg_restore -h localhost -U postgres -d vjsp_onecall /opt/DB/shop.dump  #备份

pg_dump -h localhost -W -U postgres -f /opt/DB/shop.sql VJSP20004611 #还原

```

