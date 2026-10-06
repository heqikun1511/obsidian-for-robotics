

cd src/<pkg>

git remote add origin git@gitlab.xmutros2snake.com:xmut/ros_packages/<name>.git

git fetch origin # 先看远端有什么，别急着 branch -M

git checkout -b feature/arti origin/main # 或 origin/master，接住远端的初始 commit

git add -A && git commit -m "..."

git push -u origin feature/arti # 新分支，无需 -f