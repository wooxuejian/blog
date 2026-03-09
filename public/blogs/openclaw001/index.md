# 使用开源三件套OpenClaw+Ollama+1Panel部署7×24运行的个人AI助理
## 一、写在前面
本次操作教程将以开源 Linux 服务器运维面板 1Panel 为基础，搭配 Ollama 本地大模型（无需担心 Token 消耗费用），手把手教你部署 OpenClaw 个人 AI 助理，实现 7×24 小时稳定运行，轻松拥有专属智能助手！

## 二、资源准备
本次 OpenClaw 本地个人 AI 助理基于一台腾讯 GPU 云服务器构建，云服务器获取过程可参见腾讯云官网，服务器具体配置如下：
- 操作系统：Ubuntu Server 24.04 LTS 64 位
- 计算资源：20 核 80 G
- 磁盘容量：100G
- GPU：计算型 GN7 | GN7.5XLARGE80
- 网络：绑定弹性公网IP

## 三、操作过程
基于 1Panel 服务器运维管理面板构建本地化 AI 助理大致分为以下五个步骤：
1. 1Panel 安装部署；
2. GPU 资源调度配置；
3. Ollama 的安装部署；
4. Qwen3 模型加载；
5. OpenClaw 安装部署及配置。

## 四、详细操作步骤说明
### 4.1 1Panel 安装部署
1Panel 安装可参照官网在线安装：[https://1panel.cn/docs/v2/installation/online_installation/](https://1panel.cn/docs/v2/installation/online_installation/)
#### 步骤一：获取 root 权限
登录服务器后切换到 root 权限：
```bash
sudo su -
```
#### 步骤二：执行在线安装命令
```bash
bash -c "$(curl -sSL https://resource.fit2cloud.com/1panel/package/v2/quick_start.sh)"
```
#### 步骤三：Docker 安装
跟随安装引导，指定安装目录（默认为/opt），并确认安装 Docker（输入y），系统将自动在线安装 Docker 并选择延迟最低的源。
#### 步骤四：镜像加速器配置与面板参数设置
1. 配置镜像加速器：安装完成后选择配置（输入y），系统将自动添加腾讯云内网镜像加速配置并重启 Docker 服务；
2. 设置面板端口：可使用默认15297或自行定义（示例为26680），系统将自动打开对应防火墙端口；
3. 设置安全入口、面板用户：可使用默认值或自定义，为 Web 端访问的验证参数。
#### 步骤五：获取 1Panel 面板登录信息
安装完成后，系统会输出如下关键登录信息（示例）：
- 外部地址：http://139.186.147.190:26680/3739c8b9ad
- 内部地址：http://172.30.0.3:26680/3739c8b9ad
- 面板用户：1ade9ec6fb
- 面板密码：f969ec9c27

**注意**：云服务器需在安全组中打开对应面板端口。
#### 步骤六：验证 1Panel 部署成功
将外部地址输入浏览器进入登录页面，输入面板用户和密码，成功登录即代表安装完成。
#### 步骤七：1Panel 访问地址设置
进入面板后，切换到「面板设置」，将**默认访问地址**设置为服务器的公网IP，方便后续应用跳转和容器访问。

### 4.2 GPU 资源调度配置
进入1Panel的「终端」管理完成 NVIDIA 容器镜像配置，使容器化模型可调度 GPU 资源。
#### 步骤一：NVIDIA 显卡驱动确认
输入以下命令验证驱动是否安装成功：
```bash
nvidia-smi
```
若能正常打印出 GPU 型号、驱动版本、CUDA 版本等信息，则代表驱动安装成功；若未安装，可前往英伟达官网下载安装：[https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#installing-with-apt](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#installing-with-apt)
#### 步骤二：安装 NVIDIA 容器镜像
依次执行以下命令，为 Docker 容器配置 GPU 加速支持：
```bash
# 命令行一：添加 NVIDIA 容器工具仓库与签名
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 命令行二：启用仓库中的 experimental 组件（可选）
sed -i -e '/experimental/ s/^#//g' /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 命令行三：更新软件源
sudo apt-get update

# 命令行四：安装 nvidia-container-toolkit
sudo apt-get install -y nvidia-container-toolkit
```
#### 步骤三：配置 Docker 镜像使用 NVIDIA
执行以下命令配置 Docker 并重启服务，使 Docker 支持 NVIDIA GPU 调度：
```bash
# 配置 Docker 以使用 NVIDIA
sudo nvidia-ctk runtime configure --runtime=docker

# 重启 docker
sudo systemctl restart docker
```

### 4.3 Ollama 安装部署
Ollama 是开源的大型语言模型服务，提供类 OpenAI 的 API 接口和聊天界面，基于1Panel应用商店可一键安装。
#### 步骤一：安装 Ollama 应用
进入1Panel 的「应用商店」，点击「AI」分类，找到 **Ollama**，点击「安装」。
#### 步骤二：设置 Ollama 安装参数
配置时需注意：
- 确认版本号及默认端口11434；
- 勾选「端口外部访问」（云服务器需确保安全组开通11434端口）；
- 勾选「开启 GPU 支持」（确保服务器有 NVIDIA GPU 且驱动已安装）；
- 其他参数保持默认，点击「确认」。
#### 步骤三：下载镜像并安装 Ollama
点击确认后，系统将自动拉取 Ollama 镜像并启动应用，直至提示「安装应用[ollama]成功」即完成。
#### 步骤四：验证 Ollama 是否成功
进入「已安装应用」，找到 Ollama，点击「跳转」，若页面输出 **Ollama is running**，则代表部署成功。

### 4.4 Qwen3 本地模型部署
基于已部署的 Ollama，在1Panel中加载 Qwen3 模型（示例为qwen3:14b）。
#### 步骤一：创建 Qwen3 模型
进入1Panel的「AI」-「模型管理」页面，点击「添加模型」。
#### 步骤二：加载模型
1. 前往 Ollama 官网（[ollama.com/library/qwen3](ollama.com/library/qwen3)）查找模型 ID；
2. 在「名称」中输入模型 ID（示例：qwen3:14b），点击「添加」开始加载；
3. 模型加载耗时约20-30分钟，具体取决于服务器网络和硬件。
#### 步骤三：模型运行确认
当模型列表中的「状态」更新为「成功」后，点击「运行」，若能与模型正常对话，则代表模型加载成功。

### 4.5 OpenClaw 安装部署
OpenClaw 为开源自托管的个人 AI 助手，基于已部署的 Ollama 和 Qwen3 模型，在1Panel中快速安装配置。
#### 步骤一：开始安装 OpenClaw 应用
进入1Panel「应用商店」-「AI」分类，找到 **OpenClaw**，点击「安装」，进入参数设置页面。
#### 步骤二：设置应用安装参数
关键参数配置说明（其余保持默认）：
| 参数项 | 配置要求 |
| ---- | ---- |
| 端口 | 默认为18789（WebUI）、18790（Bridge），若端口被占用可调整，确保安全组开通 |
| 模型供应商 | 下拉选择「Ollama」 |
| 模型 | 按格式输入：Ollama/模型ID（示例：Ollama/qwen3:14b） |
| 模型API Key | 本地模型无需真实密钥，输入任意字符即可 |
| Base URL | 输入 Ollama 访问地址+`/v1`（示例：http://139.186.147.190:11434/v1） |
| 端口外部访问 | 勾选，方便外部访问 OpenClaw |

配置完成后点击「确认」开始安装。
#### 步骤三：OpenClaw 应用安装
点击确认后，系统自动拉取 OpenClaw 镜像并启动应用，直至提示「安装应用[openclaw]成功」即完成。
#### 步骤四：应用 token 获取
1. 进入「已安装应用」，找到 OpenClaw，点击「进入目录」；
2. 找到 `data/conf` 文件夹，打开 `openclaw.json` 文件；
3. 找到「gateway」中的「token」字段，复制对应的 token 值（示例：9bfd07dd800a8c304b62bfac09f698cb7ad9f939d812021a）。
#### 步骤五：Web 应用访问设置
1. 进入 OpenClaw 的「参数配置」页面；
2. 将「Web 访问地址」拼接为以下格式：`http://服务器IP:WebUI端口?token=复制的token值`；
3. 示例：http://139.186.147.190:18789?token=9bfd07dd800a8c304b62bfac09f698cb7ad9f939d812021a，保存配置。

## 五、个人 AI 助理效果
完成所有配置后，进入1Panel「已安装应用」，找到 OpenClaw，点击「跳转」（选择带 token 的链接），即可进入 OpenClaw 控制台使用个人 AI 助理。

OpenClaw 可实现**网页信息抓取、智能对话、轻量办公辅助**等功能，示例：输入「帮我在腾讯云网站找下相关云计算服务内容」，模型可自动抓取腾讯云官网内容并整理反馈。

## 六、总结
1. **部署优势**：以 1Panel 为核心载体，所有配置可视化、步骤化，无需复杂命令行功底，新手也能快速完成；本地化部署无需依赖外部 Token，数据隐私更有保障，可实现7×24小时不间断运行。
2. **模型选择建议**：基于本次硬件配置（20核80G+NVIDIA GPU），**Qwen3:14b、Qwen coder:30b** 是兼顾性能与资源消耗的优选；若服务器显存、算力更充足，可尝试部署更大参数模型以获得更优效果。
3. **现存局限与展望**：当前本地模型在工具调用的灵活性上仍有不足，但随着 OpenClaw、Ollama 等开源项目的持续迭代，功能将不断优化，为工作和生活效率带来显著提升。

这套开源三件套（OpenClaw+Ollama+1Panel）搭建的本地化 AI 助理方案，兼具实用性、安全性和可操作性，适合个人和小型团队搭建专属智能助手。