# 1. 每台机器只需执行一次，安装 LFS 钩子

git lfs install

  

# 2. 进入目标仓库，声明要追踪的模式（大小写敏感，两个都写）

cd src/snake_description

git lfs track "*.STL" "*.stl"

  

# 3. 这一步会生成/更新 .gitattributes —— 它必须被提交，否则别人 clone 后规则失效

git add .gitattributes

  

# 4. 正常提交文件

git add .

git commit -m "Add STL meshes via Git LFS"

git push origin <branch>





`git lfs ls-files # 列出当前被 LFS 跟踪的文件`

`git lfs status # 查看哪些文件会走 LFS`

`git lfs env # 检查 LFS 端点配置`

`git lfs pull # 拉取真实的 LFS 内容（clone 后如果没有自动下载）`

`git lfs fsck # 校验 LFS 对象完整性`


