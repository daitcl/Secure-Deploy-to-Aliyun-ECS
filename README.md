- ## 使用GitHub Actions 安全部署到阿里云 ECS

  ------

  ### 1. 概述

  - **目标**：在 GitHub Actions 运行部署任务时，自动获取运行器的公网 IP，通过阿里云 API 将该 IP 临时加入安全组（仅允许访问 22 端口），部署完成后立即删除该规则，从而避免长期开放端口带来的安全风险。
  - **优势**：避免长期开放 22 端口，大幅降低被扫描和暴力破解的风险。
  - **适用场景**：需要从 GitHub Actions 安全地 SSH 登录阿里云 ECS 进行部署的任何项目。
  
  ------
  
  ### 2. 准备工作：阿里云端配置
  
  #### 2.1 确认 ECS 实例和安全组信息
  
  1. 登录阿里云控制台，进入 **ECS 管理页面**。
  2. 找到目标实例，记录以下信息（后续需要填入 GitHub Secrets）：
     - **实例所属地域 ID**（例如 `cn-hangzhou`，详见第 5 节表格）
     - **实例绑定的安全组 ID**（例如 `sg-xxxxxx`）
  3. 进入该安全组的 **入方向规则** 页面，**删除所有允许 `0.0.0.0/0` 访问 22 端口的规则**。目前可以保持为空，或仅保留内网访问规则。
  
  #### 2.2 创建 RAM 子账号并授权
  
  为 GitHub Actions 创建一个专用的 RAM 子用户，并授予管理安全组的权限。
  
  1. 访问 [RAM 控制台](https://ram.console.aliyun.com/)。
  2. 创建用户：
     - **登录名称**：例如 `github-actions`
     - **访问配置**：勾选 **使用永久 AccessKey 访问**
     - **AccessKey**：勾选 **我确认必须创建 AccessKey**（系统将生成 AccessKey ID 和 Secret）
  3. **保存 AccessKey ID 和 Secret**（务必立即保存，关闭后将无法再次查看）。
  4. 为该用户授权（**新增授权**）：
     - 添加权限 **AliyunECSFullAccess**（系统策略，允许管理 ECS 资源，包括安全组）。
     - 若需更精细的权限，可参考 8.3 节自定义策略。
  
  ------
  
  ### 3. GitHub 仓库配置 Secrets
  
  在 GitHub 仓库的 **Settings → Secrets and variables → Actions** 中添加以下敏感信息：
  
  | Secret 名称                | 说明                                      |
  | :------------------------- | :---------------------------------------- |
  | `ALIYUN_ACCESS_KEY_ID`     | 上一步创建的 RAM 用户的 AccessKey ID      |
  | `ALIYUN_ACCESS_KEY_SECRET` | 对应的 AccessKey Secret                   |
  | `ALIYUN_REGION`            | ECS 实例所在地域 ID（例如 `cn-hangzhou`） |
  | `ALIYUN_SECURITY_GROUP_ID` | 安全组 ID（例如 `sg-xxxxxx`）             |
  
  > 如果你还需要通过 SSH 部署，请额外添加以下 Secrets（可选）：
  >
  > - `REMOTE_HOST`：ECS 公网 IP 或域名
  > - `REMOTE_USER`：SSH 用户名
  > - `SSH_PRIVATE_KEY`：SSH 私钥
  
  ------
  
  ### 4. 通用 GitHub Actions 工作流模板
  
  将以下代码片段放入你的工作流文件中（如 `.github/workflows/your-workflow.yml`），并在 `# 在这里插入你的构建/部署步骤` 处替换为你自己的任务。
  
  ```yaml
  name: Secure Deploy to Aliyun ECS
  
  on:
    push:
      branches: [ main ]   # 触发分支，可按需修改
  
  jobs:
    deploy:
      runs-on: ubuntu-latest
      steps:
        # ---------- 1. 获取当前运行器的公网 IP ----------
        - name: Get Runner Public IP
          id: ip
          run: |
            RUNNER_IP=$(curl -s http://ifconfig.me)
            echo "CURRENT_IP=$RUNNER_IP" >> $GITHUB_OUTPUT
            echo "Runner IP: $RUNNER_IP"
  
        # ---------- 2. 安装阿里云 CLI（二进制方式）----------
        - name: Install Aliyun CLI
          run: |
            curl -LO "https://aliyuncli.alicdn.com/aliyun-cli-linux-latest-amd64.tgz"
            tar -xzf aliyun-cli-linux-latest-amd64.tgz
            mkdir -p $HOME/.local/bin
            mv aliyun $HOME/.local/bin/
            echo "$HOME/.local/bin" >> $GITHUB_PATH
            aliyun --version
  
        # ---------- 3. 临时放行当前 IP（22 端口）----------
        - name: Authorize Security Group (Add IP)
          env:
            ALIBABA_CLOUD_ACCESS_KEY_ID: ${{ secrets.ALIYUN_ACCESS_KEY_ID }}
            ALIBABA_CLOUD_ACCESS_KEY_SECRET: ${{ secrets.ALIYUN_ACCESS_KEY_SECRET }}
            ALIBABA_CLOUD_REGION_ID: ${{ secrets.ALIYUN_REGION }}
          run: |
            aliyun ecs AuthorizeSecurityGroup \
              --RegionId ${{ secrets.ALIYUN_REGION }} \
              --SecurityGroupId ${{ secrets.ALIYUN_SECURITY_GROUP_ID }} \
              --IpProtocol tcp \
              --PortRange 22/22 \
              --SourceCidrIp ${{ steps.ip.outputs.CURRENT_IP }}/32 \
              --Policy accept \
              --NicType intranet
  
        # （可选）等待几秒让规则生效
        - name: Wait for rule to take effect
          run: sleep 5
  
        # ---------- 在这里插入你的构建/部署步骤 ----------
        # 例如：
        # - name: Build and Deploy
        #   run: |
        #     echo "Your deployment commands here"
        #   # 或使用 SSH Action：
        # # ---------- 验证 SSH 访问成功 ----------
        # - name: Verify SSH Connection
        #   uses: appleboy/ssh-action@v1.0.3
        #   with:
        #     host: ${{ secrets.REMOTE_HOST }}          # ECS 公网 IP 或域名
        #     username: ${{ secrets.REMOTE_USER }}      # 登录用户名（如 root 或普通用户）
        #     key: ${{ secrets.SSH_PRIVATE_KEY }}       # 私钥内容（需在 secrets 中配置）
        #     port: 22                                   # SSH 端口，默认 22
        #     script: |
        #       echo "✅ SSH connection to ECS is successful!"
        #       echo "Hostname: $(hostname)"
        #       echo "Current user: $(whoami)"
        #       echo "Working directory: $(pwd)"
        #       # 可以在此处添加更多验证命令，如检查必要目录是否存在
  
        # ---------- 4. 撤销放行（删除刚才添加的规则）----------
        - name: Revoke Security Group (Remove IP)
          if: always()
          env:
            ALIBABA_CLOUD_ACCESS_KEY_ID: ${{ secrets.ALIYUN_ACCESS_KEY_ID }}
            ALIBABA_CLOUD_ACCESS_KEY_SECRET: ${{ secrets.ALIYUN_ACCESS_KEY_SECRET }}
            ALIBABA_CLOUD_REGION_ID: ${{ secrets.ALIYUN_REGION }}
          run: |
            aliyun ecs RevokeSecurityGroup \
              --RegionId ${{ secrets.ALIYUN_REGION }} \
              --SecurityGroupId ${{ secrets.ALIYUN_SECURITY_GROUP_ID }} \
              --IpProtocol tcp \
              --PortRange 22/22 \
              --SourceCidrIp ${{ steps.ip.outputs.CURRENT_IP }}/32 \
              --NicType intranet
  ```
  
  
  
  ------
  
  ### 5. 阿里云地域与可用区对照表
  
  在配置 `ALIYUN_REGION` 时，请根据你的 ECS 实例所在地域选择对应的**地域 ID**（例如 `cn-hangzhou`）。
  
  | 地域名称                    | 地域ID         |
  | :-------------------------- | :------------- |
  | 华北1（青岛）               | cn-qingdao     |
  | 华北2（北京）               | cn-beijing     |
  | 华北3（张家口）             | cn-zhangjiakou |
  | 华北5（呼和浩特）           | cn-huhehaote   |
  | 华北6（乌兰察布）           | cn-wulanchabu  |
  | 华东1（杭州）               | cn-hangzhou    |
  | 华东2（上海）               | cn-shanghai    |
  | 华东5（南京-本地地域）      | cn-nanjing     |
  | 华东6（福州-本地地域）      | cn-fuzhou      |
  | 华中1（武汉-本地地域）      | cn-wuhan-lr    |
  | 华南1（深圳）               | cn-shenzhen    |
  | 华南2（河源）               | cn-heyuan      |
  | 华南3（广州）               | cn-guangzhou   |
  | 西南1（成都）               | cn-chengdu     |
  | 中国香港                    | cn-hongkong    |
  | 新加坡                      | ap-southeast-1 |
  | 澳大利亚（悉尼）            | ap-southeast-2 |
  | 马来西亚（吉隆坡）          | ap-southeast-3 |
  | 印度尼西亚（雅加达）        | ap-southeast-5 |
  | 菲律宾（马尼拉）            | ap-southeast-6 |
  | 泰国（曼谷）                | ap-southeast-7 |
  | 印度（孟买）                | ap-south-1     |
  | 日本（东京）                | ap-northeast-1 |
  | 韩国（首尔）                | ap-northeast-2 |
  | 美国（硅谷）                | us-west-1      |
  | 美国（弗吉尼亚）            | us-east-1      |
  | 德国（法兰克福）            | eu-central-1   |
  | 英国（伦敦）                | eu-west-1      |
  | 阿联酋（迪拜）              | me-east-1      |
  | 沙特（利雅得-合作伙伴运营） | me-central-1   |
  
  > **注意**：安全组授权时只需要地域 ID，无需指定具体的可用区。
  
  ------
  
  ### 6. 步骤详解
  
  #### 步骤 1：获取运行器 IP
  
  - 使用 `ifconfig.me` 服务获取当前 GitHub Actions 运行器的公网出口 IP。
  - 将 IP 保存到 `steps.ip.outputs.CURRENT_IP` 中，供后续步骤使用。
  - 也可使用其他服务如 `ip.sb`、`icanhazip.com`。
  
  #### 步骤 2：安装阿里云 CLI
  
  - 直接下载官方最新版 CLI 二进制文件，解压后放入 `PATH`，避免依赖第三方 Action，更稳定可靠。
  
  #### 步骤 3：添加安全组规则
  
  - 调用 `AuthorizeSecurityGroup` 接口，将当前 IP 添加到安全组，协议 TCP，端口 22。
  - `--NicType intranet` 适用于 VPC 网络的实例；如果是经典网络，可能需要改为 `internet`。
  - 添加规则后等待几秒，确保规则生效（可选，但推荐）。
  
  #### 步骤 4：插入你的构建/部署步骤
  
  - 在这里执行你原本的构建、测试、SSH 部署等操作。
  - 由于安全组已临时放行，SSH 连接可以成功。
  
  #### 步骤 5：删除安全组规则
  
  - 调用 `RevokeSecurityGroup` 接口，传入与添加时完全相同的参数，精确删除规则。
  - `if: always()` 确保无论部署步骤成功或失败，都会执行清理，避免规则残留。
  
  ------
  
  ### 7. 注意事项与常见问题
  
  #### Q1：授权/撤销时返回错误 `InvalidIpProtocol.Malformed` 或 `InvalidParameter`
  
  - 检查 `--PortRange` 格式是否为 `22/22`，`--SourceCidrIp` 是否带 `/32`。
  - 确保地域 ID、安全组 ID 正确，且 RAM 用户有相应权限。
  - 确认 `--NicType` 与实例网络类型匹配（VPC 用 `intranet`，经典网络用 `internet`）。

  #### Q2：SSH 部署失败（连接超时或权限被拒）

  - 可能原因：安全组规则尚未生效。可在授权后增加 `sleep 5` 步骤。
  - 检查 ECS 实例内部防火墙（如 iptables）是否允许 22 端口，安全组只是网络层过滤。
  - 确认 SSH 服务正在运行，且公钥已正确配置到 `~/.ssh/authorized_keys`。
  
  #### Q3：撤销步骤失败，导致规则残留
  
  - 确保撤销步骤的参数与添加步骤完全一致（包括大小写、空格、掩码等）。
  - 检查 `if: always()` 是否正确添加。
  
  #### Q4：阿里云 CLI 提示权限不足
  
  - 确认 RAM 用户已授予 `AliyunECSFullAccess` 权限，并且 AccessKey 正确无误。
  
  #### Q5：安全组规则达到上限
  
  - 每个安全组默认最多添加 200 条规则（以阿里云文档为准）。如果频繁执行且删除失败导致规则堆积，可能达到上限。建议定期检查并手动清理残留规则。
  
  #### Q6：GitHub Actions IP 变更
  
  - GitHub 的 runners IP 地址范围可能变化，但每个 job 的出口 IP 是固定的。本方案动态获取当前 IP，无需预知 IP 段，因此不受影响。
  
  ------
  
  ### 8. 进阶优化
  
  #### 8.1 超时保护
  
  - 在使用 SSH Action 时，可设置 `timeout` 参数（如 `appleboy/ssh-action` 支持 `timeout`），防止部署任务长时间卡住导致规则一直开放。
  
  #### 8.2 并发处理
  
  - 多个 workflow 同时运行时，会各自添加和删除自己的 IP 规则，互不影响。但需注意安全组规则数量上限。
  
  #### 8.3 自定义 RAM 策略（最小权限）
  
  若需最小权限，可创建自定义策略，仅允许操作特定安全组。示例策略如下（需将 `cn-hangzhou` 和 `sg-xxxxxx` 替换为实际值）：
  
  ```json
  {
    "Version": "1",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "ecs:AuthorizeSecurityGroup",
          "ecs:RevokeSecurityGroup"
        ],
        "Resource": "acs:ecs:cn-hangzhou:*:securitygroup/sg-xxxxxx"
      }
    ]
  }
  ```
  
  
  
  #### 8.4 使用 OIDC 代替长期 AccessKey（高级）
  
  GitHub Actions 支持 OIDC 与阿里云 RAM 角色集成，可避免管理 AccessKey Secret，进一步提升安全性。具体配置请参考阿里云官方文档。
