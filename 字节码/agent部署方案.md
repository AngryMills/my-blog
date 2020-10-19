# agent部署方案

## 方案1（生产机器无法访问公网，pass）

1. 需要部署的机器指定目录下放置 /data/holmes-agent/update.sh

2. CICD - 部署配置 - 前置上线脚本 - /data/holmes-agent/update.sh

3. 脚本内容

   ```bash
   
   ```

   

## 方案2（借助CICD）

1. 同一个项目组共用一个agent-cicd 项目
2. 可选择是否需要发布agent
3. 可配置需要覆盖的ip(不同的项目每次发布需要修改ip)
4. 与业务服务完全独立

