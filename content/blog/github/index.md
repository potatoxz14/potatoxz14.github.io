---
title: 🎉 GitHub commit
summary: 关于提commit 代码到github上
date: 2025-08-28

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - github
---

## 建立连接

在将本地代码上传到 GitHub 之前，您需要配置 SSH 公私钥，以便进行安全的认证。以下是配置 SSH 公私钥的步骤：

1. **生成 SSH 密钥：** 打开终端或命令提示符，运行以下命令生成 SSH 密钥。如果您已经有 SSH 密钥，请跳过此步骤。

   ```
   bashCopy code
   ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
   ```

   按照提示输入文件名和密码。一般来说，使用默认文件名和留空密码是常见的选择。

2. **添加 SSH 密钥到 SSH 代理：** 运行以下命令将 SSH 密钥添加到 SSH 代理（ssh-agent）。

   ```
   bashCopy codeeval "$(ssh-agent -s)"
   ssh-add ~/.ssh/id_rsa
   ```

3. **将 SSH 公钥添加到 GitHub：**

   - 运行以下命令将公钥内容复制到剪贴板：

     ```
     bashCopy codepbcopy < ~/.ssh/id_rsa.pub  # macOS
     cat ~/.ssh/id_rsa.pub | clip  # Windows
     ```

   - 登录到 GitHub。

   - 在右上角找到头像，选择 "Settings"。

   - 在左侧导航栏选择 "SSH and GPG keys"。

   - 点击 "New SSH key" 或 "Add SSH key"。

   - 将剪贴板中的公钥粘贴到 "Key" 字段中，并为密钥起一个描述性的标题。

   - 点击 "Add SSH key"。

4. **验证 SSH 连接：** 运行以下命令验证是否可以成功连接到 GitHub。

   ```
   bashCopy code
   ssh -T git@github.com
   ```

# git clone之后上传

要将 `git clone` 之后做的修改上传回 GitHub，你需要执行以下步骤：

1. **在 GitHub 上创建仓库：**

   - 登录 GitHub 账户。
   - 在右上角找到加号（+）并选择 "New repository"。
   - 输入仓库名称、描述和其他选项，然后点击 "Create repository"。

2. **在本地初始化 Git 仓库：**

   - 打开终端或命令提示符。

   - 进入要上传的本地项目的目录，使用 `cd` 命令。

   - 运行以下命令初始化本地 Git 仓库：

     ```
     bashCopy code
     git init
     ```

3. **将本地项目添加到 Git 仓库：**

   - 运行以下命令将项目所有文件添加到 Git 仓库：

     ```
     bashCopy code
     git add .
     ```

4. **提交更改：**

   - 运行以下命令提交更改到本地仓库，其中 "initial commit" 是提交的注释，您可以根据需要修改：

     ```
     bashCopy code
     git commit -m "initial commit"
     ```

5. **关联本地仓库到 GitHub 仓库：**

   - 在 GitHub 仓库页面找到 "Quick setup" 部分，复制其中的远程仓库地址。

   - 返回到终端，运行以下命令关联本地仓库到 GitHub 仓库（将 `<repository-url>` 替换为实际的 GitHub 仓库地址）：

     ```
     git remote add origin <repository-url>
     ```

6. **推送本地更改到 GitHub：**

   - 运行以下命令将本地更改推送到 GitHub（如果是第一次推送，需要使用 `-u` 选项，将本地的 `master` 分支关联到远程的 `origin` 仓库的 `master` 分支）：

     ```
     git push -u origin master
     ```

# 建立分支

1.查看所有分支

```
git branch
```

2.更新分支

```
git fetch --all
```

3.更换分支

```
git checkout  <new_branch>
```

4.创建分支

```
git branch <new_branch>
```



# 打包成patch

```
git diff '原来的' '新的' > test.patch

 git diff origin/6.2.7 . > test.patch
```

# 方法：使用 Git 强制重写历史

1. **Clone 仓库**

   如果你没有本地副本，请克隆你的 GitHub 仓库：

   ```
   bash复制代码git clone https://github.com/username/repository.git
   cd repository
   ```

2. **创建一个新的 orphan 分支**

   这个分支不会有任何历史记录，等于一个全新的初始化分支。

   ```
   bash
   
   
   复制代码
   git checkout --orphan new-branch
   ```

   `--orphan` 会创建一个没有历史记录的分支。

3. **添加文件**

   将所有当前文件添加到暂存区：

   ```
   bash
   
   
   复制代码
   git add .
   ```

4. **创建一个新的初始 commit**

   现在为这个干净的分支创建一个新的 commit：

   ```
   bash
   
   
   复制代码
   git commit -m "Initial commit"
   ```

5. **删除旧的 `master/main` 分支**

   在此步骤中，将删除旧的 `master` 或 `main` 分支（取决于你的默认分支名称），并将新分支重命名为默认分支：

   ```
   bash复制代码git branch -D main  # 如果你的默认分支是 main
   git branch -D master  # 如果你的默认分支是 master
   ```

6. **将新分支重命名为 `main` 或 `master`**

   ```
   bash复制代码git branch -m main  # 如果你的默认分支是 main
   git branch -m master  # 如果你的默认分支是 master
   ```

7. **强制推送到 GitHub**

   此步骤将覆盖 GitHub 上的历史记录。请确保此操作不会影响其他协作开发者，因为历史记录将被完全重写。

   ```
   bash复制代码git push --force origin main  # 如果你的默认分支是 main
   git push --force origin master  # 如果你的默认分支是 master
   ```

8. **删除旧的分支引用（可选）**

   有时 GitHub 上可能还有旧分支的引用，例如 `refs/original/`。你可以删除这些：

   ```
   bash
   
   
   复制代码
   git reflog expire --expire=now --all && git gc --prune=now --aggressive
   ```

------

### 结果

完成上述步骤后，所有的 commit 历史都会从 GitHub 仓库中删除，仓库将只有一个新的初始 commit，像是全新的仓库。

### 注意事项

- **不可逆操作**：这个操作是不可逆的，尤其在协作项目中，所有的 commit 历史都将丢失，其他开发者无法再基于旧历史进行开发。
- **破坏性操作**：强制推送 (`--force`) 会覆盖远程仓库，务必确保这不会影响正在开发的其他用户。

如果这是一个公开仓库或多人协作项目，请确保所有相关人员都清楚这一操作。
