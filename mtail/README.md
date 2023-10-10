## Usage

```
ansible-playbook -i hosts site.yml
```

### 检测hosts连通性

```
ansible -i hosts all -m ping
```

### 启动服务

```
ansible -i hosts all -m shell -a "cd /app/loan/mtail && nohup ./mtail -port 3903 -progs ./hj.mtail -logs \"/app/logs/prometheus/*/*.log\" -vm_logs_runtime_errors=false >/dev/null 2>&1 & "
```

